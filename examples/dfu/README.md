# DFU example
@anchor dfu_example

@tag52810nosupport

This Device Firmware Update (DFU) example illustrates how to create an application that can be updated
over the mesh using background mode DFU. In this mode, the new firmware is transferred in the background
while the application is running and the DFU reports to the application when the transfer is done.
The application can then flash the new firmware when ready.

More information about the DFU can be found in the @ref md_doc_libraries_dfu_dfu_protocol section.

---

## Software requirements @anchor dfu_example_requirements_hw

This DFU application requires a bootloader and a valid device page to function correctly.

---

## Setup @anchor dfu_example_setup

You can find the source code of the DFU example in the following folder: `<InstallFolder>/examples/dfu`

Precompiled bootloaders are included in the `bin` directory at the project root directory
and the device page must be generated with the `device_page_generator.py` script located in `tools/dfu/device_page.py`.
See also @ref md_tools_dfu_README.

---

## Testing the example @anchor dfu_example_testing

See @ref md_doc_libraries_dfu_dfu_quick_start for detailed instructions on how to perform
a DFU operation using this example application.

