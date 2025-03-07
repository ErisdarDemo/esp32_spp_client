# ESP-IDF SPP GATT CLIENT demo

Two Demos, a server & client that connect and exchange data wirelessly with each other, using a BLE SPP

- See ble_spp_server\README.md for more definition & description

## Setup

- server: WROVER-Kit	(ESP32 - COM6 UART)

	Dev: VS Code

- client: Waveshare 	(ESP32-S3 - COM4 JTAG)	

	Dev: Espressif IDE

## Procedure

@pre Boards unplugged, IDEs closed
	
- Connect boards & validate COM ports

- Open both IDEs
	
- Open Espressif IDE client debug session halted at app_main()
	
- Open PuTTy to client serial port
	
- Launch VS Code server flash, open IDF Monitor 'Monitor Device' & confirm running 
	
- Resume Espressif debug session

## Development Opens

- Get Client UART Rx working (debug with LabVIEW? Why is it not catching the PuTTy tx?)

- Get both serial streams open & working

### Hardware Required

* A development board with ESP32/ESP32-C3/ESP32-S3/ESP32-C2/ESP32-H2 SoC (e.g., ESP32-DevKitC, ESP-WROVER-KIT, etc.)
* A USB cable for Power supply and programming

See [Development Boards](https://www.espressif.com/en/products/devkits) for more information about it.

## How to Use Example

Before project configuration and build, be sure to set the correct chip target using:

```bash
idf.py set-target <chip_name>
```

### Initialization

1. Both the server and client will first initialize the uart and ble

- The server demo will set up the serial port service with standard GATT and GAP services in the attribute server

- The client demo will scan the ble broadcast over the air to find the spp server

### Event Processing

  The spp server has two main event processing functions for BLE event:

```c
  void gap_event_handler(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t * param);
  void gatts_profile_event_handler(esp_gatts_cb_event_t event, esp_gatt_if_t gatts_if, esp_ble_gatts_cb_param_t * param);
```

  The spp client has two main event processing functions for BLE event:

```c
  esp_gap_cb(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t * param);
  void gattc_profile_event_handler(esp_gattc_cb_event_t event, esp_gatt_if_t gattc_if, esp_ble_gattc_cb_param_t * param);
```

  These are some queues and tasks used by SPP application:

  Queues:

  * spp_uart_queue       - Uart data messages received from the Uart
  * cmd_cmd_queue        - commands received from the client
  * cmd_heartbeat_queue  - heartbeat received, if supported

  Tasks:

  * `uart_task`            - process Uart
  * `spp_cmd_task`         - process command messages, the commands and processing were defined by customer
  * `spp_heartbeat_task`   - if heartbeat is supported, the task will send a heartbeat packet to the remote device

### Packet Structure

  After the Uart received data, the data will be posted to Uart task. Then, in the UART_DATA event, the raw data may be retrieved. The max length is 120 bytes every time.
  If you run the ble spp demo with two ESP32 chips, the MTU size will be exchanged for 200 bytes after the ble connection is established, so every packet can be send directly.
  If you only run the ble_spp_server demo, and it was connected by a phone, the MTU size may be less than 123 bytes. In such a case the data will be split into fragments and send in turn.
  In every packet, we add 4 bytes to indicate that this is a fragment packet. The first two bytes contain "##" if this is a fragment packet, the third byte is the total number of the packets, the fourth byte is the current number of this packet.
  The phone APP need to check the structure of the packet if it want to communicate with the ble_spp_server demo.

### Sending Data Wirelessly

  The client will be sending WriteNoRsp packets to the server. The server side sends data through notifications. When the Uart receives data, the Uart task places it in the buffer. If the size of the data is larger than (MTU size - 3), the data will be split into packets and send in turn.

### Receiving Data Wirelessly

  The server will receive this data in the ESP_GATTS_WRITE_EVT event and send data to the Uart terminal by `uart_write_bytes` function. For example:

    case ESP_GATTS_WRITE_EVT:
            ...
        if(res == SPP_IDX_SPP_DATA_RECV_VAL){
            uart_write_bytes(UART_NUM_0, (char *)(p_data->write.value), p_data->write.len);
        }
            ...
    break;

### GATT Server Attribute Table

  characteristic|UUID|Permissions
  :-:|:-:|:-:
  SPP_DATA_RECV_CHAR|0xABF1|READ&WRITE_NR
  SPP_DATA_NOTIFY_CHAR|0xABF2|READ&NOTIFY
  SPP_COMMAND_CHAR|0xABF3|READ&WRITE_NR
  SPP_STATUS_CHAR|0xABF4|READ & NOTIFY
  SPP_HEARTBEAT_CHAR|0xABF5|READ&WRITE_NR&NOTIFY

### Build and Flash

Run `idf.py -p PORT flash monitor` to build, flash and monitor the project.

(To exit the serial monitor, type ``Ctrl-]``.)

See the [Getting Started Guide](https://idf.espressif.com/) for full steps to configure and use ESP-IDF to build projects.

## Example Output

The spp client will auto connect to the spp server, do service search, exchange MTU size and register notification.

### Client

```
I (2894) GATTC_SPP_DEMO: ESP_GATTC_CONNECT_EVT: conn_id=0, gatt_if = 3
I (2894) GATTC_SPP_DEMO: REMOTE BDA:
I (2904) GATTC_SPP_DEMO: 00 00 00 00 00 00
I (2904) GATTC_SPP_DEMO: EVT 2, gattc if 3
I (3414) GATTC_SPP_DEMO: EVT 7, gattc if 3
I (3414) GATTC_SPP_DEMO: ESP_GATTC_SEARCH_RES_EVT: start_handle = 40, end_handle = 65535, UUID:0xabf0
I (3424) GATTC_SPP_DEMO: EVT 6, gattc if 3
I (3424) GATTC_SPP_DEMO: SEARCH_CMPL: conn_id = 0, status 0
I (3464) GATTC_SPP_DEMO: EVT 18, gattc if 3
I (3464) GATTC_SPP_DEMO: +MTU:200

I (3464) GATTC_SPP_DEMO: attr_type = PRIMARY_SERVICE,attribute_handle=40,start_handle=40,end_handle=65535,properties=0x0,uuid=0xabf0
I (3474) GATTC_SPP_DEMO: attr_type = CHARACTERISTIC,attribute_handle=42,start_handle=0,end_handle=0,properties=0x6,uuid=0xabf1
I (3484) GATTC_SPP_DEMO: attr_type = CHARACTERISTIC,attribute_handle=44,start_handle=0,end_handle=0,properties=0x12,uuid=0xabf2
I (3494) GATTC_SPP_DEMO: attr_type = DESCRIPTOR,attribute_handle=45,start_handle=0,end_handle=0,properties=0x0,uuid=0x2902
I (3504) GATTC_SPP_DEMO: attr_type = CHARACTERISTIC,attribute_handle=47,start_handle=0,end_handle=0,properties=0x6,uuid=0xabf3
I (3524) GATTC_SPP_DEMO: attr_type = CHARACTERISTIC,attribute_handle=49,start_handle=0,end_handle=0,properties=0x12,uuid=0xabf4
I (3534) GATTC_SPP_DEMO: attr_type = DESCRIPTOR,attribute_handle=50,start_handle=0,end_handle=0,properties=0x0,uuid=0x2902
I (3544) GATTC_SPP_DEMO: Index = 2,UUID = 0xabf2, handle = 44
I (3554) GATTC_SPP_DEMO: EVT 38, gattc if 3
I (3554) GATTC_SPP_DEMO: Index = 2,status = 0,handle = 44
I (3594) GATTC_SPP_DEMO: EVT 9, gattc if 3
I (3594) GATTC_SPP_DEMO: ESP_GATTC_WRITE_DESCR_EVT: status =0,handle = 45
I (3654) GATTC_SPP_DEMO: Index = 5,UUID = 0xabf4, handle = 49
I (3654) GATTC_SPP_DEMO: EVT 38, gattc if 3
I (3654) GATTC_SPP_DEMO: Index = 5,status = 0,handle = 49
I (3684) GATTC_SPP_DEMO: EVT 9, gattc if 3
I (3684) GATTC_SPP_DEMO: ESP_GATTC_WRITE_DESCR_EVT: status =0,handle = 50
I (16904) GATTC_SPP_DEMO: EVT 10, gattc if 3
I (16904) GATTC_SPP_DEMO: ESP_GATTC_NOTIFY_EVT
I (16904) GATTC_SPP_DEMO: +NOTIFY:handle = 44,length = 22
```

### Server

```
I (4452) GATTS_SPP_DEMO: EVT 14, gatts if 3
I (4452) GATTS_SPP_DEMO: event = e
I (5022) GATTS_SPP_DEMO: EVT 4, gatts if 3
I (5022) GATTS_SPP_DEMO: event = 4
I (5152) GATTS_SPP_DEMO: EVT 2, gatts if 3
I (5152) GATTS_SPP_DEMO: event = 2
I (5152) GATTS_SPP_DEMO: ESP_GATTS_WRITE_EVT : handle = 5
I (5242) GATTS_SPP_DEMO: EVT 2, gatts if 3
I (5242) GATTS_SPP_DEMO: event = 2
I (5242) GATTS_SPP_DEMO: ESP_GATTS_WRITE_EVT : handle = 10
I (18462) GATTS_SPP_DEMO: EVT 5, gatts if 3
I (18462) GATTS_SPP_DEMO: event = 5
I (27652) GATTS_SPP_DEMO: EVT 2, gatts if 3
I (27652) GATTS_SPP_DEMO: event = 2
I (27652) GATTS_SPP_DEMO: ESP_GATTS_WRITE_EVT : handle = 2
```
if you input data to the Uart terminal, it will be printed in the remote device Uart terminal.

## Troubleshooting

For any technical queries, please open an [issue](https://github.com/espressif/esp-idf/issues) on GitHub. We will get back to you soon.

