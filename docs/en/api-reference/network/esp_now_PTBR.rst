ESP-NOW
=======

:link_to_translation:`zh_CN:[中文]`

Visão geral
-----------

 
ESP-NOW é um tipo de protocolo de comunicação Wi-Fi sem conexão definido pela Espressif. No ESP-NOW, os dados do aplicativo são encapsulados em um quadro de ação específico do fornecedor e depois transmitidos de um dispositivo Wi-Fi para outro sem conexão.

CTR com protocolo CBC-MAC (CCMP) é usado para proteger o quadro de ação para segurança. ESP-NOW é amplamente utilizado em luz inteligente, controle remoto, sensor, etc.



Formato do Pacote
-----------------

Usa um quadro de dados específico do fornecedor para transmitir dados ESP-NOW. A taxa de bits padrão do ESP-NOW é de 1 Mbps. O formato do pacote específico do fornecedor é o seguinte:


 .. highlight:: table

::
 
 | --------- | --------- | -------------- | --------- | ----------- | ------- |
 | Cabecalho |  Codigo   |  Identificador |   Valor   |  Conteudo   |   FCS   |
 |    MAC    | Categoria | da Organização | Aleatório | especifico  |         |
 | --------- | --------- | -------------- | --------- | ----------- | ------- |
 |  24 bytes |  1 byte   |    3 bytes     |  4 bytes  | 7-257 bytes | 4 bytes |
 | --------- | --------- | -------------- | --------- | ----------- | ------- |
  
- Código da categoria: O campo Código da categoria é definido com o valor (127) que indica a categoria específica do fornecedor.
- Identificador da Organização: O Identificador da Organização contém um identificador único (0x18fe34), que são os três primeiros bytes do endereço MAC aplicado pelo Espressif.
- Valor Aleatório: O campo Valor Aleatório é usado para evitar ataques de retransmissão.
- Conteúdo específico do fornecedor: o Conteúdo específico do fornecedor contém campos específicos do fornecedor, como segue:

.. highlight:: nenhum

::

    -------------------------------------------------------------------------------
    | Element ID | Length | Organization Identifier | Type |  Versao |   Corpo    |
    -------------------------------------------------------------------------------
        1 byte     1 byte            3 bytes         1 byte   1 byte   0-250 bytes

- Element ID: O campo Element ID é definido com o valor (221), indicando o elemento específico do fornecedor.
- Comprimento: O comprimento é o comprimento total do Identificador, Tipo, Versão e Corpo da Organização.
- Organization Identifier: O Identificador da Organização contém um identificador único (0x18fe34), que são os três primeiros bytes do endereço MAC aplicado pelo Espressif.
- Tipo: O campo Tipo é definido com o valor (4) indicando ESP-NOW.
- Versão: O campo Versão é definido como a versão do ESP-NOW.
- Corpo: O Corpo contém os dados do ESP-NOW.

Como o ESP-NOW não tem conexão, o cabeçalho MAC é um pouco diferente dos frames padrão. Os bits FromDS e ToDS do campo FrameControl são ambos 0. O primeiro campo de endereço é definido como o endereço de destino. O segundo campo de endereço é definido como o endereço de origem. O terceiro campo de endereço está definido para endereço de transmissão (0xff:0xff:0xff:0xff:0xff:0xff).

Segurança
---------

ESP-NOW usa o método CCMP, descrito no padrão IEEE 802.11-2012, para proteger o quadro de ação específico do fornecedor. O dispositivo Wi-Fi mantém uma chave mestra primária (PMK) e várias chaves mestras locais (LMK). Os comprimentos de PMK e LMk são 16 bytes.

     * PMK é usado para criptografar LMK com o algoritmo AES-128. Chame :cpp:func:`esp_now_set_pmk()` para definir PMK. Se o PMK não estiver definido, um PMK padrão será usado.
     * O LMK do dispositivo emparelhado é usado para criptografar o quadro de ação específico do fornecedor com o método CCMP. O número máximo de LMKs diferentes é seis. Se o LMK do dispositivo emparelhado não estiver definido, o quadro de ação específico do fornecedor não será criptografado.

A criptografia de quadro de ação multicast específico do fornecedor não é suportada.

Inicialização e desinicialização
--------------------------------

Chamar a função :cpp:func:`esp_now_init()` para inicializar o ESP-NOW e :cpp:func:`esp_now_deinit()` para desinicializar o ESP-NOW. Os dados do ESP-NOW devem ser transmitidos após o início do Wi-Fi, por isso é recomendado iniciar o Wi-Fi antes de inicializar o ESP-NOW e parar o Wi-Fi após desinicializar o ESP-NOW.

Quando :cpp:func:`esp_now_deinit()` é chamado, todas as informações dos dispositivos emparelhados são deletadas.

Adicionar dispositivo emparelhado
---------------------------------

Chamar :cpp:func:`esp_now_add_peer()` para adicionar o dispositivo à lista de dispositivos emparelhados antes de enviar dados para este dispositivo. Se a segurança estiver ativada, o LMK deverá ser definido. Você pode enviar dados ESP-NOW através da estação e da interface SoftAP. Certifique-se de que a interface esteja habilitada antes de enviar dados ESP-NOW.

.. Somente:: esp32c2

    O número máximo de dispositivos emparelhados é 20 e os dispositivos de criptografia emparelhados não são superiores a 4, o padrão é 2. Se você deseja alterar o número de dispositivos de criptografia emparelhados, defina :ref:`CONFIG_ESP_WIFI_ESPNOW_MAX_ENCRYPT_NUM` no componente Wi-Fi menu de configuração.

.. Somente:: esp32 ou esp32s2 ou esp32s3 ou esp32c3 ou esp32c6

O número máximo de dispositivos emparelhados é 20, e os dispositivos de criptografia emparelhados não são superiores a 17, o padrão é 7. Se você deseja alterar o número de dispositivos de criptografia emparelhados, defina :ref:`CONFIG_ESP_WIFI_ESPNOW_MAX_ENCRYPT_NUM` no componente Wi-Fi menu de configuração.

A device with a broadcast MAC address must be added before sending broadcast data. The range of the channel of paired devices is from 0 to 14. If the channel is set to 0, data will be sent on the current channel. Otherwise, the channel must be set as the channel that the local device is on.

Enviando Dados ESP-NOW
----------------------

Chamar :cpp:func:`esp_now_send()` para enviar dados ESP-NOW e :cpp:func:`esp_now_register_send_cb()` para registrar uma funçãi callback de envio. Retornando  `ESP_NOW_SEND_SUCCESS` na função de Callback de envio se o dado for recebido
com sucesso na camada MAC. Caso contrário, retornará `ESP_NOW_SEND_FAIL`. Vários motivos podem fazer com que o ESP-NOW falhe no envio de dados. Por exemplo, o dispositivo de destino não existe; os canais dos dispositivos não são iguais; o quadro de ação é perdido durante a transmissão no ar, etc. Não é garantido que a camada de aplicação possa receber os dados. Se necessário, envie dados de confirmação ao receber dados ESP-NOW. Se o tempo limite dos dados de confirmação for recebido, retransmita os dados ESP-NOW. Um número de sequência também pode ser atribuído aos dados ESP-NOW para eliminar os dados duplicados.

Se houver muitos dados ESP-NOW para enviar, chame :cpp:func:`esp_now_send()` para enviar menos ou igual a 250 bytes de dados uma vez por vez. Observe que um intervalo muito curto entre o envio de dois dados ESP-NOW pode causar desordem no envio da função de retorno de chamada. Portanto, é recomendado que o envio dos próximos dados do ESP-NOW após o retorno da função de retorno de chamada do envio anterior. A função de retorno de chamada de envio é executada a partir de uma tarefa Wi-Fi de alta prioridade. Portanto, não faça operações demoradas na função de retorno de chamada. Em vez disso, publique os dados necessários em uma fila e trate-os em uma tarefa de prioridade mais baixa.

Recebendo Dados ESP-NOW
-----------------------

Chame :cpp:func:`esp_now_register_recv_cb()` para registrar a função de retorno de chamada recebida. Chame a função de callback(retorno) de chamada de recebimento ao receber ESP-NOW. A função de retorno de chamada de recebimento também é executada na tarefa Wi-Fi. Portanto, não faça operações demoradas na função de retorno de chamada.
Em vez disso, publique os dados necessários em uma fila e trate-os em uma tarefa de prioridade mais baixa.

Configurar taxa ESP-NOW
-----------------------

.. somente:: esp32 ou esp32s2 ou esp32s3 ou esp32c2 ou esp32c3

Chame :cpp:func:`esp_wifi_config_espnow_rate()` para configurar a taxa ESP-NOW da interface especificada. Certifique-se de que a interface esteja habilitada antes da taxa de configuração. Esta API deve ser chamada após :cpp:func:`esp_wifi_start()`.


.. somente:: esp32c6

Chame :cpp:func:`esp_now_set_peer_rate_config()` para configurar a taxa ESP-NOW de cada peer. Certifique-se de que o par seja adicionado antes de configurar a taxa. Esta API deve ser chamada após :cpp:func:`esp_wifi_start()` e :cpp:func:`esp_now_add_peer()`.

    .. nota::

        :cpp:func:`esp_wifi_config_espnow_rate()` está obsoleto, por favor use cpp::func:`esp_now_set_peer_rate_config()` em seu lugar.

Configurar parâmetro de economia de energia ESP-NOW
--------------------------------------------

A suspensão é suportada apenas quando {IDF_TARGET_NAME} está configurado como estação.

Chame :cpp:func:`esp_now_set_wake_window()` para configurar o Window para ESP-NOW RX durante o sono. O valor padrão é o máximo, permitindo RX o tempo todo.

Se a economia de energia for necessária para ESP-NOW, chame :cpp:func:`esp_wifi_connectionless_module_set_wake_interval()` para configurar o intervalo também.

.. apenas:: SOC_WIFI_SUPPORTED

     Consulte :ref:`economia de energia do módulo sem conexão <connectionless-module-power-save>` para obter mais detalhes.

Exemplos de aplicação
---------------------

* Exemplo de envio e recebimento de dados ESP-NOW entre dois dispositivos: :exemplo:`wifi/espnow`.

* Para obter mais exemplos de aplicação de como usar o ESP-NOW, visite o repositório `ESP-NOW <https://github.com/espressif/esp-now>`_.

Referência de API
-----------------


.. include-build-file:: inc/esp_now.inc
