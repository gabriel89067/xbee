

Utilizando o **modo API**, com envio de **frames 0x10** e recepção via **frames 0x90**.

---

## ✅ Endereços de 64 bits (SH + SL)

| Nome | SH     | SL       | Endereço 64-bit           |
| ---- | ------ | -------- | ------------------------- |
| ESP1 | 13A200 | 426BE1E3 | `00 13 A2 00 42 6B E1 E3` |
| ESP2 | 13A200 | 426BE1E4 | `00 13 A2 00 42 6B E1 E4` |
| ESP3 | 13A200 | 426BE1B6 | `00 13 A2 00 42 6B E1 B6` |

---

## ✅ Função de Envio (frame 0x10)

```c
void send_api_message(const uint8_t dest_addr[8], const char *msg) {
    uint8_t frame[128];
    int i = 0;

    frame[i++] = 0x7E;

    uint16_t data_len = 14 + strlen(msg);
    frame[i++] = (data_len >> 8) & 0xFF;
    frame[i++] = data_len & 0xFF;

    frame[i++] = 0x10; // Frame type
    frame[i++] = 0x01; // Frame ID

    memcpy(&frame[i], dest_addr, 8); i += 8;

    frame[i++] = 0xFF; // 16-bit addr unknown
    frame[i++] = 0xFE;

    frame[i++] = 0x00; // Broadcast radius
    frame[i++] = 0x00; // Options

    int msg_len = strlen(msg);
    memcpy(&frame[i], msg, msg_len); i += msg_len;

    uint8_t sum = 0;
    for (int j = 3; j < i; j++) sum += frame[j];
    frame[i++] = 0xFF - sum;

    uart_write_bytes(UART_NUM, (const char *)frame, i);
    ESP_LOGI("XBee", "Enviado: %s", msg);
}
```

---

## ✅ Função Receptora (frame 0x90)

```c
void task_receive_xbee(void *arg) {
    uint8_t buf[128];
    int len = 0;

    while (1) {
        len = uart_read_bytes(UART_NUM, buf, sizeof(buf), 100 / portTICK_PERIOD_MS);
        if (len > 0 && buf[0] == 0x7E) {
            uint16_t length = (buf[1] << 8) | buf[2];
            if (length + 4 > len) continue;

            uint8_t frame_type = buf[3];
            if (frame_type == 0x90) {
                char sender[20];
                snprintf(sender, sizeof(sender), "%02X%02X%02X%02X%02X%02X%02X%02X",
                         buf[4], buf[5], buf[6], buf[7], buf[8], buf[9], buf[10], buf[11]);

                uint8_t *data = &buf[15];
                int data_len = length - 12;
                char msg[100] = {0};
                memcpy(msg, data, data_len);
                msg[data_len] = '\0';

                ESP_LOGI("XBee", "De %s: %s", sender, msg);
            }
        }
    }
}
```

---

## ✅ `app_main()` no ESP1 – Envia para ESP2 e depois para ESP3

```c
void app_main() {
    uart_init();
    xTaskCreate(task_receive_xbee, "xbee_rx", 4096, NULL, 10, NULL);

    // Endereço do ESP2
    uint8_t addr_esp2[8] = {0x00, 0x13, 0xA2, 0x00, 0x42, 0x6B, 0xE1, 0xE4};
    // Endereço do ESP3
    uint8_t addr_esp3[8] = {0x00, 0x13, 0xA2, 0x00, 0x42, 0x6B, 0xE1, 0xB6};

    vTaskDelay(pdMS_TO_TICKS(3000)); // Espera inicial

    // Envia para ESP2
    send_api_message(addr_esp2, "Oi 2, sou o 1");
    vTaskDelay(pdMS_TO_TICKS(2000));

    // Envia para ESP3
    send_api_message(addr_esp3, "Oi 3, sou o 1");
}
```

---

## ✅ ESP2 e ESP3

Ambos usam o mesmo `task_receive_xbee()` e `uart_init()`, e **não precisam enviar nada nesse exemplo** — apenas receber e exibir a mensagem.

Você pode depois configurar para que eles **respondam automaticamente**, se quiser.

---

## ✅ Pronto para testar!

Com isso:

* O **ESP1** envia mensagens para o ESP2 e depois para o ESP3, usando API mode via XBee.
* O **ESP2** e **ESP3** recebem, interpretam e exibem o conteúdo.

---

Se quiser, posso:

* ✅ Adicionar **resposta automática** no ESP2 e ESP3
* 🔄 Montar um esquema de **"ping-pong"**
* 📡 Adicionar **broadcast** do ESP1 para todos

Deseja expandir com alguma dessas opções?
