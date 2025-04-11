import ctypes as C
import os
import time

class Spectrometer:
    def __init__(self,
                 serial_number,
                 name='CCS100',
                 reset=True,
                 verbose=True,
                 very_verbose=True):
        self.scan_data = None  # initialize，avoid AttributeError
        self.name = name
        self.verbose = verbose
        self.very_verbose = very_verbose
        if self.verbose: print("%s: opening..."%self.name)
        resourceName = ("USB0::0x1313::0x8081::" + serial_number + "::RAW").encode('ascii')
        self.handle = C.c_uint32()
        dll.init(resourceName, 1, reset, self.handle) # IDQuery = True (default)
        if self.verbose: print("%s: open and ready."%self.name)
        self._get_device_info()
        self._get_wavelength_data()
        self._get_status()
        self.get_integration_time()

    def _get_device_info(self):
        if self.verbose:
            print("%s: getting device info:"%self.name)
        device_info = [(256 * C.c_char)() for i in range(5)] # 5 entries
        dll.get_device_info(self .handle, *device_info)
        for i, info in enumerate(device_info):
            device_info[i] = info.value.decode('ascii')
        if self.verbose:
            print("%s:  - manufacturer  = %s"%(self.name, device_info[0]))
            print("%s:  - device        = %s"%(self.name, device_info[1]))
            print("%s:  - serial_number = %s"%(self.name, device_info[2]))
            print("%s:  - firmware      = %s"%(self.name, device_info[3]))
            print("%s:  - driver        = %s"%(self.name, device_info[4]))
        self.device_info = tuple(device_info)
        return self.device_info

    def _get_wavelength_data(self, user_calibration=False):
        if self.very_verbose:
            print("%s: getting wavelength data"%self.name)
        wave_data = (3648 * C.c_double)()
        wave_min  = C.c_double()
        wave_max  = C.c_double()
        dll.get_wavelength_data(
            self.handle, user_calibration, wave_data, wave_min, wave_max)
        self.wavelength_data = tuple(wave_data)
        self.wavelength_min  = wave_min.value
        self.wavelength_max  = wave_max.value
        if self.very_verbose:
            print("%s: wavelength_min (nm) = %s (s)"%(
                self.name, self.wavelength_min))
            print("%s: wavelength_max (nm) = %s (s)"%(
                self.name, self.wavelength_max))
        return self.wavelength_data

    def _get_status(self):
        if self.very_verbose:
            print("%s: getting status"%self.name)
        self.status = C.c_int32()
        dll.get_status(self.handle, self.status)
        # waiting for software or external trigger to start scan:
        self.status_idle_soft_trig      = bool(self.status.value & 0x0002)
        self.status_idle_ext_trig       = bool(self.status.value & 0x0080)
        # scan starting or in progress:
        self.status_scan_starting       = bool(self.status.value & 0x0008)
        self.status_scan_in_progress    = bool(self.status.value & 0x0004)
        # scan is done and ready for data transfer:
        self.status_scan_ready          = bool(self.status.value & 0x0010)
        if self.very_verbose:
            print("%s: -> status_idle_soft_trig = %s"%(
                self.name, self.status_idle_soft_trig))
            print("%s: -> status_idle_ext_trig = %s"%(
                self.name, self.status_idle_ext_trig))
            print("%s: -> status_scan_starting = %s"%(
                self.name, self.status_scan_starting))
            print("%s: -> status_scan_in_progress = %s"%(
                self.name, self.status_scan_in_progress))
            print("%s: -> status_scan_ready = %s"%(
                self.name, self.status_scan_ready))
        return self.status

    def _start_scan(self):
        if self.very_verbose:
            print("%s: starting scan"%self.name)
        dll.start_scan(self.handle)
        if self.very_verbose:
            print("%s: -> done starting scan."%self.name)
        return None

    def _get_scan_data(self):
        if self.very_verbose:
            print("%s: getting scan data"%self.name)
        scan_data = (3648 * C.c_double)()
        dll.get_scan_data(self.handle, scan_data)
        self.scan_data =  tuple(scan_data)
        if self.very_verbose:
            print("%s: -> done getting scan data."%self.name)
        return self.scan_data

    def get_integration_time(self):
        if self.verbose:
            print("%s: getting integration time"%self.name)
        integration_time = C.c_double()
        dll.get_integration_time(self.handle, integration_time)
        self.integration_time_s = integration_time.value
        if self.verbose:
            print("%s:  = %s (s)"%(self.name, self.integration_time_s))
        return self.integration_time_s

    def set_integration_time(self, integration_time_s):
        if self.verbose:
            print("%s: setting integration time (s) = %s"%(
                self.name, integration_time_s))
        assert (isinstance(integration_time_s, int)
                or isinstance(integration_time_s, float)), (
                    "%s: unexpected type for integration_time_s"%self.name)
        assert 1e-5 <= integration_time_s <= 6e1, (
            "%s: integration_time_s (%s) out of range"%(
                self.name, integration_time_s))
        dll.set_integration_time(self.handle, integration_time_s)
        self.get_integration_time()
        tol_s = 1e-2 # 10ms tolerance on actual integration time
        assert self.integration_time_s <= integration_time_s + tol_s
        assert self.integration_time_s >= integration_time_s - tol_s
        if self.verbose:
            print("%s: -> done setting integration time."%self.name)
        return None

    def get_spectrum(self, filename=None):
        if self.verbose:
            print("%s: getting spectrum"%self.name)
        self._get_status()

        #Sometimes, for some special cases, like if you interrupt the scanning process when CCS100 is running, you need to reset the equipment, otherwise it will report error
        retry_count = 5
        wait_time = 2

        while not self.status_idle_soft_trig and retry_count > 0:
            print("%s: -> status_idle_soft_trig = %s" % (
                self.name, self.status_idle_soft_trig))
            print("%s: -> spectrometer not ready to scan with software" %
                  self.name)
            print("Attempting reset...")
            dll.reset(self.handle)
            time.sleep(wait_time)
            self._get_status()
            retry_count -= 1

        if not self.status_idle_soft_trig:
            print("Error: Spectrometer is still not ready for software trigger.")
            self.scan_data = None
            return None

        self._start_scan()

        if self.verbose:
            print("%s: -> waiting for scan..."%self.name, end='')
        while not self.status_scan_ready:
            if self.verbose:
                print(".", end='')
            time.sleep(self.integration_time_s / 10)
            self._get_status()
        if self.verbose:
            print("\n%s: -> scan ready"%self.name)
        self._get_scan_data()
        if self.very_verbose:
            print("%s: -> done getting spectrum."%self.name)
        if filename is not None:
            if self.verbose:
                print("%s: saving spectrum (%s)"%(self.name, filename))
            with open(filename + '.txt', 'w') as file:
                for wave, data in zip(self.wavelength_data, self.scan_data):
                    file.write('%0.3f'%wave + ':' + str(data) + '\n')
        return self.wavelength_data, self.scan_data

    def plot_spectrum(self,
                      wavelength_data=None,
                      scan_data=None,
                      show=True,
                      filename=None):
        import matplotlib.pyplot as plt
        if self.verbose:
            print("%s: plotting spectrum"%self.name)
        if wavelength_data is None:
            if self.verbose:
                print("%s: (using last recorded wavelength data)"%self.name)
            wavelength_data = self.wavelength_data
        if scan_data is None:
            if self.verbose:
                print("%s: (using last recorded scan data)"%self.name)
            scan_data = self.scan_data
        fig, ax = plt.subplots()
        ax.set_title('Spectrum (integration_time_s = %0.3f)'%
                     self.integration_time_s)
        ax.set_ylabel('Intensity (a.u.)')
        ax.set_xlabel('Wavelength (nm)')
        ax.plot(wavelength_data, scan_data)
        ax.grid(True)
        ax.set_ylim(-1, 1)
        ax.set_xlim(min(wavelength_data), max(wavelength_data))
        if show:
            if self.verbose:
                print("%s: showing plot (matplotlib window)"%self.name)
            plt.show()
        if filename is not None:
            if self.verbose:
                print("%s: saving plot (%s)"%(self.name, filename))
            fig.savefig(filename, dpi=150)
        plt.close(fig)
        if self.verbose:
            print("%s: -> done plotting spectrum."%self.name)
        return None

    def close(self):
        if self.verbose: print("%s: closing..."%self.name, end='')
        dll.close(self.handle)
        if self.verbose: print(" done.")
        return None

os.add_dll_directory(os.getcwd())
dll = C.cdll.LoadLibrary("TLCCS_64.dll") # needs "TLCCS_64.dll" in directory

def check_error(error_code):
    if error_code != 0:
        print("Error message from Thorlabs CCS100: ", end='')
        raise UserWarning(
            "Thorlabs CCS100 error: %i; see above for details."%(error_code))
    return error_code

dll.init = dll.tlccs_init
dll.init.argtypes = [
    C.c_char_p,                 # resourceName
    C.c_bool,                   # IDQuery
    C.c_bool,                   # resetDevice
    C.POINTER(C.c_uint32)]      # instrumentHandle
dll.init.restype = check_error

dll.get_device_info = dll.tlccs_identificationQuery
dll.get_device_info.argtypes = [
    C.c_uint32,                 # instrumentHandle
    C.c_char_p,                 # _VI_FAR manufacturerName[]
    C.c_char_p,                 # _VI_FAR deviceName[]
    C.c_char_p,                 # _VI_FAR serialNumber[]
    C.c_char_p,                 # _VI_FAR firmwareRevision[]
    C.c_char_p]                 # _VI_FAR instrumentDriverRevision[]
dll.get_device_info.restype = check_error

dll.get_wavelength_data = dll.tlccs_getWavelengthData
dll.get_wavelength_data.argtypes = [
    C.c_uint32,                 # instrumentHandle
    C.c_int16,                  # dataSet
    3648 * C.c_double,          # _VI_FAR wavelengthDataArray[]
    C.POINTER(C.c_double),      # minimumWavelength
    C.POINTER(C.c_double)]      # maximumWavelength
dll.get_wavelength_data.restype = check_error

dll.get_status = dll.tlccs_getDeviceStatus
dll.get_status.argtypes = [
    C.c_uint32,                 # instrumentHandle
    C.POINTER(C.c_int32)]       # deviceStatus
dll.get_status.restype = check_error

dll.start_scan = dll.tlccs_startScan
dll.start_scan.argtypes = [
    C.c_uint32]                 # instrumentHandle
dll.start_scan.restype = check_error

dll.get_scan_data = dll.tlccs_getScanData
dll.get_scan_data.argtypes = [
    C.c_uint32,                 # instrumentHandle
    3648 * C.c_double]          # _VI_FAR data[]
dll.get_scan_data.restype = check_error

dll.get_integration_time = dll.tlccs_getIntegrationTime
dll.get_integration_time.argtypes = [
    C.c_uint32,                 # instrumentHandle
    C.POINTER(C.c_double)]      # integrationTimes
dll.get_integration_time.restype = check_error

dll.set_integration_time = dll.tlccs_setIntegrationTime
dll.set_integration_time.argtypes = [
    C.c_uint32,                 # instrumentHandle
    C.c_double]                 # integrationTimes
dll.set_integration_time.restype = check_error

dll.close = dll.tlccs_close
dll.close.argtypes = [
    C.c_uint32]                 # instrumentHandle
dll.close.restype = check_error
dll.reset = dll.tlccs_reset

if __name__ == '__main__':
    spec = Spectrometer(
        serial_number="M01216839", verbose=True, very_verbose=False)

    print('\n# Set integration time:')
    spec.set_integration_time(0.2)

    print('\n# Scan and plot:')
    spec.get_spectrum()
    spec.plot_spectrum()

    print('\n# Saving:')
    filename='example'
    spec.get_spectrum(filename=filename)
    spec.plot_spectrum(show=False, filename=filename)

    spec.close()
