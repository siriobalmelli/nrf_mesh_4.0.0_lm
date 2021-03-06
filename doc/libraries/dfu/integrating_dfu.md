# Integrating DFU process into the application

Once you are familiar with the integral parts of the @ref md_doc_libraries_dfu_dfu_protocol,
you can start integrating DFU into your application.

This page describes the stages that your application must include to make the DFU process work,
and in particular the crucial API functions that are required.

After including these stages in your application, go to @ref md_doc_libraries_dfu_dfu_quick_start
to prepare your environment for performing the DFU update and ensuring that the DFU process coded
into the application works as intended.

**Table of contents**
- [Initialization](@ref integrating_dfu_init)
- [Updating firmware](@ref integrating_dfu_update)
    - [Receiving packets](@ref integrating_dfu_update_receive_packets)
    - [Requesting new firmware](@ref integrating_dfu_update_request_fmw)
        - [Selecting a bank address](@ref integrating_dfu_update_request_fmw_bank_address)
    - [Receiving new firmware](@ref integrating_dfu_update_receive_fmw)
    - [Verifying new firmware](@ref integrating_dfu_update_verify)
    - [Flashing new firmware](@ref integrating_dfu_update_flash)
- [Relaying firmware](@ref integrating_dfu_relay)
- [Aborting transfer](@ref integrating_dfu_abort)


---

## Initialization @anchor integrating_dfu_init

The Mesh DFU module is initialized and enabled by the mesh stack, when  
the application calls the functions @ref mesh_stack_init and @ref mesh_stack_start, respectively. 
No more actions are required on the application side.

See the following figure for an overview of the initialization process.

![DFU initialization](img/dfu_process_init.svg)

@note The @ref nrf_mesh_dfu_init will work correctly only after you [flash the serial bootloader on all devices](@ref dfu_configuration_flash_bl),
which happens as part of @ref md_doc_libraries_dfu_dfu_quick_start.

Once initialized, the Mesh DFU module switches to the @ref NRF_MESH_DFU_STATE_INITIALIZED 
state and starts advertising [Firmware ID](@ref dfu-packet-fwid) packets with 
the current firmware IDs.

---

## Updating firmware @anchor integrating_dfu_update

All DFU packets are received by the mesh stack (the scanner) and passed to the Mesh DFU module by
the @ref nrf_mesh_dfu_rx function. This process does not require any actions from the application.

@note While flash operations are in progress, the Mesh DFU module is unable to
receive packets.

![Firmware update process](img/dfu_process_receive.svg)

### Receiving packets @anchor integrating_dfu_update_receive_packets

Upon receiving a [Firmware ID](@ref dfu-packet-fwid) packet or a [DFU state](@ref dfu-packet-state)
packet, the Mesh DFU module notifies the application about a new firmware using 
@ref NRF_MESH_EVT_DFU_FIRMWARE_OUTDATED (or @ref NRF_MESH_EVT_DFU_FIRMWARE_OUTDATED_NO_AUTH) events.
This happens before the firmware transfer begins, and the Mesh DFU module will only notify
the application if it has not started the reception of the firmware.

Depending on the the parameters supplied in @ref nrf_mesh_evt_dfu_t.fw_outdated, the application 
may decide to either:
    - Receive new firmware if the version of the application in the supplied parameters is higher
    than the one installed.
    - Relay the packets from other relay nodes or target.
    
If neither @ref nrf_mesh_dfu_request nor @ref nrf_mesh_dfu_relay is called as a 
result of a @ref NRF_MESH_EVT_DFU_FIRMWARE_OUTDATED
(or @ref NRF_MESH_EVT_DFU_FIRMWARE_OUTDATED_NO_AUTH) event, the update is ignored.

### Requesting new firmware @anchor integrating_dfu_update_request_fmw
    
If the application decides to receive new firmware, it must call the @ref nrf_mesh_dfu_request
function with the following elements specified:
    - DFU type @ref nrf_mesh_dfu_type_t,
    - Firmware ID @ref nrf_mesh_fwid_t, and
    - an address where to store of the banked data.
    See [the following subsection](@ref integrating_dfu_update_request_fmw_bank_address)
    for details about how to select the address.

Once @ref nrf_mesh_dfu_request is called, the Mesh DFU module switches
to the [target role](@ref dfu-protocol-roles).
    
#### Selecting a bank address @anchor integrating_dfu_update_request_fmw_bank_address

The bank address decides where the firmware image is to be stored while the 
transfer is in progress.

Make the application choose a bank address that has enough space for the entire firmware image,
without overlapping with the final location of the transferred firmware. Since the size info for
the incoming firmware image is not available to the device in the firmware ID packet, a good idea 
is to leave as much space as possible for the bank and the resulting firmware location.
This can be achieved by placing the bank address in the middle of the application 
area, at the equal distance from the start of the application and from the mesh persistent 
storage location (which can be found by calling @ref mesh_stack_persistence_flash_usage).
This calculation is demonstrated in the @ref md_examples_dfu_README.

### Receiving new firmware @anchor integrating_dfu_update_receive_fmw

The process of receving [DFU data](@ref dfu-packet-data) packets is handled by 
the Mesh DFU module in the background.

The Mesh DFU module stores the received firmware in the bank at the provided address, which
overrides the data stored at that address.

### Verifying new firmware @anchor integrating_dfu_update_verify

When the last segment is received, the Mesh DFU module verifies the signature of the 
received firmware.
- If the signature validation fails, the Mesh DFU module notifies the application
by sending @ref NRF_MESH_EVT_DFU_END with the reason @ref NRF_MESH_DFU_END_ERROR_UNAUTHORIZED.
- If the signature validation succeeds, the Mesh DFU module proceeds to the flash operations,
and the application receives @ref NRF_MESH_EVT_DFU_END event
with the status @ref NRF_MESH_DFU_END_SUCCESS.

For more information about signing the firmware, see [Security](@ref dfu-protocol-security)
on the Mesh DFU protocol page.

### Flashing new firmware @anchor integrating_dfu_update_flash

Once the DFU process ends successfully and the bank is ready, the application receives
@ref NRF_MESH_EVT_DFU_BANK_AVAILABLE with the type of the bank (@ref nrf_mesh_evt_dfu_t::bank).

The application can now call @ref nrf_mesh_dfu_bank_flash to replace the current firmware 
with the new one. This process reboots the device. After the update, the bootloader
switches to the application.

@note Flashing new firmware triggers a restart of the chip. All non-volatile memory is 
lost during this call.


---

## Relaying firmware @anchor integrating_dfu_relay

Upon receiving @ref NRF_MESH_EVT_DFU_FIRMWARE_OUTDATED (or @ref NRF_MESH_EVT_DFU_FIRMWARE_OUTDATED_NO_AUTH) event
and depending on the the parameters supplied in @ref nrf_mesh_evt_dfu_t.fw_outdated,
the application can ask the Mesh DFU module to relay [DFU data](@ref dfu-packet-data) packets
coming from other relay nodes or target. This can happens, for example, when the application node
has the latest version of firmware already installed.

To relay packets, the application must call the @ref nrf_mesh_dfu_relay function and provide 
the DFU type @ref nrf_mesh_dfu_type_t and the Firmware ID @ref nrf_mesh_fwid_t of the firmware to
be relayed. See the following figure.

![Relaying firmware](img/dfu_process_relay.svg)

The application can switch the Mesh DFU module to the [relay role](@ref dfu-protocol-roles)
by calling the @ref nrf_mesh_dfu_relay function. The application can call this function
upon receiving the @ref NRF_MESH_EVT_DFU_FIRMWARE_OUTDATED
(or @ref NRF_MESH_EVT_DFU_FIRMWARE_OUTDATED_NO_AUTH) event, or at any other time.

When the relaying process is started, the Mesh DFU module switches to 
@ref NRF_MESH_DFU_STATE_RELAY state and sends @ref NRF_MESH_EVT_DFU_START event
to the application.

If no packets are received during 60 seconds, the Mesh DFU module notifies the application
by sending @ref NRF_MESH_EVT_DFU_END event with the reason @ref NRF_MESH_DFU_END_ERROR_TIMEOUT.
The Mesh DFU module is switched to the @ref NRF_MESH_DFU_STATE_INITIALIZED
state and the application must call @ref nrf_mesh_dfu_relay to start relaying the DFU data packets
again.

---

## Aborting transfer @anchor integrating_dfu_abort

The application can abort the ongoing DFU transfer by calling @ref nrf_mesh_dfu_abort.
The Mesh DFU module is then switched to the @ref NRF_MESH_DFU_STATE_INITIALIZED
state.
