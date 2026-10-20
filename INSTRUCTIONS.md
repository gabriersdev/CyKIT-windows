### Organização do Projeto CyKit

Após clonar ou extrair o repositório, a estrutura do projeto é distribuída da seguinte forma:

```text
CyKIT-master/
├── Examples/                  # Scripts de exemplo para consumo direto (ex.: example_epoc_plus.py)
├── Web/                       # Interface gráfica para navegadores
│   ├── CyKIT.html             # Painel de controle e visualizador gráfico (EEG, Gyro, gravação)
│   ├── CySocketClient.js      # Cliente WebSocket que se comunica com o CyKIT.py
│   └── Themes/                # Estilos e temas visuais da interface
└── Py3/                       # NÚCLEO DO SISTEMA (onde a execução deve ocorrer)
    ├── CyKIT.py               # Script principal / ponto de entrada da aplicação
    ├── eeg.py                 # Módulo de descriptografia AES, filtros e buffers
    ├── CyWebSocket.py         # Servidor de rede (WebSocket e TCP socket bruto)
    ├── cyDrivers/             # DLLs nativas de baixo nível
    │   ├── libusb-1.0x32.dll  # Driver USB para Python 32-bit
    │   ├── libusb-1.0x64.dll  # Driver USB para Python 64-bit
    │   ├── EEGBtleLib32.dll   # Biblioteca Bluetooth LE 32-bit
    │   └── EEGBtleLib64.dll   # Biblioteca Bluetooth LE 64-bit
    ├── cyCrypto/              # Cifras criptográficas AES Rijndael customizadas
    ├── cyPyUSB/               # Backend PyUSB adaptado com libusb
    └── cyPyWinUSB/            # Backend nativo Windows HID (alternativa à libusb)
```

---

### Regra Crítica de Execução: Por que entrar na pasta `Py3`?

> [!IMPORTANT]
> **Nunca execute o comando a partir da pasta raiz** (ex.: `python .\Py3\CyKIT.py`). É **obrigatório** navegar primeiro
para dentro do diretório `Py3` com `cd CyKIT-master\Py3`.

**Motivo técnico:**
No arquivo `eeg.py`, a chamada para carregar o driver USB foi escrita com caminho relativo ao diretório atual de
trabalho (*current working directory*):

```python
backend = cyPyUSB.backend.libusb1.get_backend(
    find_library=lambda x: "./cyDrivers/libusb-1.0x" + str(arch) + ".dll"
)
```

Se o terminal estiver na pasta raiz `CyKIT-master`, o Python procurará `./cyDrivers/...` na raiz (onde a pasta não
existe), resultando no erro imediato:
`> Driver could not be found or unsuccessfully loaded.`

**Não é necessário reiniciar o computador** para carregar essas DLLs; basta entrar na pasta `Py3` antes de executar o
script.

---

### 1. Checklist Atualizado de Passos para Receber Dados do EEG

#### Passo 1: Pareamento do Hardware e Dongle USB

1. Carregue o headset completamente.
2. Ligue o headset:
    - **EPOC**: LED azul aceso.
    - **EPOC+ / Insight**: LED branco aceso.
3. Conecte o dongle USB universal (2.4 GHz) no computador e verifique o padrão dos LEDs:
    - **Esquerdo piscando a cada segundo / Direito apagado**: Sem sinal (não pareado).
    - **Ambos acesos fixos (sem piscar)**: Dongle detectou sinal, mas não sincronizou. **Ação obrigatória:** Não
      desligue o headset; desplugue o dongle da porta USB e conecte-o novamente com o headset já ligado.
    - **Direito aceso fixo / Esquerdo fraco e oscilando (flickering)**: Conexão estabelecida com sucesso e recebendo
      pacotes.
    - *(LED vermelho indica porta/dongle danificado ou falha de hardware)*.

> **Ajuste de taxa no EPOC+:** Caso precise mudar para 256 Hz ou ligar/desligar giroscópio (MEMS), conecte o EPOC+
> **diretamente via cabo USB** antes de iniciar a transmissão sem fio (veja a seção da Wiki abaixo).

#### Passo 2: Navegação e Preparação do Ambiente Python

- O CyKit requer **Python 3.x** (versões 3.7.x, 3.8.x ou 3.9.x; 32 ou 64 bits). Evite o Python 3.12+ devido à remoção de
  módulos legados de criptografia.
- Não utilize o comando genérico `py`. Chame o executável do Python diretamente.
- Abra o terminal e entre **obrigatoriamente** na pasta `Py3`:
  ```powershell
  cd CyKIT-master\Py3
  ```

#### Passo 3: Execução do `CyKIT.py`

A estrutura do comando é:

```powershell
python.exe .\CyKIT.py <IP> <PORTA> <MODELO> [FLAGS]
```

Exemplos recomendados:

- **Para visualização no Navegador (WebSockets):**
  *(Dica: use `pywinusb` para evitar problemas de compatibilidade com a libusb no Windows)*
  ```powershell
  python.exe .\CyKIT.py 127.0.0.1 54123 6 pywinusb
  ```

- **Para visualização direta no Terminal (sem cliente de rede):**
  ```powershell
  python.exe .\CyKIT.py 127.0.0.1 54123 6 noweb+outputdata+pywinusb
  ```

- **Para envio ao OpenViBE (TCP bruto formatado):**
  ```powershell
  python.exe .\CyKIT.py 127.0.0.1 5151 6 openvibe+generic+nocounter+noheader+nobattery+ovdelay:100+float+ovsamples:004+pywinusb
  ```

#### Passo 4: Conexão do Cliente Receptor

- **Opção A — Interface Web oficial (`CyKIT.html`):**
    1. Com o `CyKIT.py` rodando, abra o arquivo `CyKIT-master\Web\CyKIT.html` no Chrome ou Firefox.
    2. Verifique o Host (`127.0.0.1`) e a Porta (`54123`).
    3. Clique no botão **Connect**.
    4. Na aba **EEG**, visualize os sinais em tempo real.
    5. Na aba **Recording**, clique em **Record** e depois em **Stop** para gerar os dados em arquivo CSV.
- **Opção B — OpenViBE:**
    1. Abra o *OpenViBE Acquisition Server*.
    2. Selecione o driver **Generic Raw Telnet Reader**.
    3. Nas propriedades do driver: Host `localhost`, Porta `5151`, Canais `14`, Frequência `128` (ou `256`), Endianness
       `Big Endian`, Sample type `32 bits float` (com `+float`) ou `16 bits unsigned integer` (com `+integer`), skips em
       `0`.
    4. Clique em **Connect** e em seguida em **Play**.
    5. Abra o *OpenViBE Designer*, ligue o *Acquisition Client* a um *Signal Display* e inicie a execução.
- **Opção C — Script TCP próprio (Python / Node.js / C#):**
    - Execute o CyKit com as flags `generic+noheader+nocounter+nobattery+eegmode`. O script escutará conexões TCP
      normais e entregará os dados delimitados por vírgula sem overhead de handshake HTTP.

---

### 2. Síntese dos Guias da Wiki do CyKit

| Guia da Wiki                       | Resumo Técnico                                                                                                                                                                                                                                                                                                                                                   |
|:-----------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **How to Install CyKIT**           | Requer Python 3.7+ (32 ou 64 bits). Não usar o inicializador `py`. Extrair o repositório, navegar até a pasta `Py3` e testar com `python.exe .\CyKIT.py help`.                                                                                                                                                                                                   |
| **How to Pair USB device**         | Detalha os estados dos LEDs do dongle USB de 2.4 GHz e ensina a regra do "desconectar e reconectar o dongle" caso ambos os LEDs fiquem fixos sem sincronizar.                                                                                                                                                                                                    |
| **How to Change EPOC+ modes**      | Conectar o headset diretamente pelo cabo USB ao PC e ligá-lo (LED branco). Acessar `Py3` e executar `CyKIT.py 127.0.0.1 54123 6 pywinusb`. Um menu interativo com 9 opções (modos 0 a 8) permite alternar taxas entre 14-bit (128 Hz) e 16-bit (128 Hz ou 256 Hz) com ou sem giroscópio (MEMS). Ao finalizar, desconectar o cabo e reemparelhar pelo dongle USB. |
| **How to Record Data in CyKIT**    | Execução do servidor na porta `54123` e abertura do `CyKIT.html` no navegador. Conexão via WebSocket local, visualização dos canais e uso da aba *Recording* para exportar diretamente para CSV.                                                                                                                                                                 |
| **How to Stream Data to OpenViBE** | Configuração do CyKit com flags de compatibilidade OpenViBE (`openvibe+generic+nocounter+noheader+nobattery...`). No OpenViBE, utilização do driver *Generic Raw Telnet Reader* na porta definida e montagem do fluxo no *OpenViBE Designer*.                                                                                                                    |

---

### 3. Modelos de Descriptografia (`KeyModel`)

O 3º parâmetro do `CyKIT.py` seleciona a cifra AES ECB correspondente ao número de série do aparelho:

| Número | Modelo do Headset | Modo de Operação                                   |
|:------:|:------------------|:---------------------------------------------------|
| **1**  | EPOC              | Modelo Premium / Research (128 Hz, 14 canais)      |
| **2**  | EPOC              | Modelo Consumer padrão (128 Hz, 14 canais)         |
| **3**  | Insight           | Modelo Premium (128 Hz, 5 canais)                  |
| **4**  | Insight           | Modelo Consumer padrão (128 Hz, 5 canais)          |
| **5**  | EPOC+             | Modelo Premium (16-bit, 256 Hz)                    |
| **6**  | EPOC+             | Modelo Consumer padrão (16-bit, 128 Hz ou 256 Hz)  |
| **7**  | EPOC+             | Modelo Consumer emulado em modo legado 14-bit EPOC |

---

### 4. Tabela de Flags Suportadas pelo `CyKIT.py`

As flags são passadas como 4º argumento do comando:

| Flag            | Tipo          | Descrição                                                                                                                         |
|:----------------|:--------------|:----------------------------------------------------------------------------------------------------------------------------------|
| `pywinusb`      | Driver        | Força o uso da biblioteca nativa HID do Windows (`cyPyWinUSB`), ignorando a `libusb`. Altamente recomendada no Windows.           |
| `info`          | Diagnóstico   | Imprime no terminal informações do hardware HID (Product Name, Device Path, Instance ID).                                         |
| `confirm`       | Interativo    | Solicita confirmação manual no console (`Use this device? [Y]es?`) toda vez que um dispositivo é identificado.                    |
| `verbose`       | Diagnóstico   | Exibe logs detalhados do sistema: threads ativas, chave AES gerada, buffers, portas e configurações aplicadas.                    |
| `outputdata`    | Visualização  | Imprime os dados decodificados/formatados diretamente no terminal.                                                                |
| `outputraw`     | Visualização  | Imprime no terminal os pacotes binários brutos recebidos antes da descriptografia.                                                |
| `noweb`         | Operação      | Permite executar o CyKit de forma autônoma sem aguardar conexão de clientes de rede/sockets.                                      |
| `generic`       | Rede          | Cria um socket TCP padrão em vez de um servidor WebSocket (sem handshake HTTP).                                                   |
| `openvibe`      | Integração    | Ajusta a serialização de pacotes para o formato telnet aceito pelo OpenViBE Acquisition Server.                                   |
| `noheader`      | Formatação    | Remove o cabeçalho de inicialização (`CyKITv2:::Info:::...`), enviando apenas dados puros. Obrigatório para OpenViBE.             |
| `nocounter`     | Formatação    | Remove os primeiros bytes de controle (`[COUNTER]` e `[INTERPOLATE]`) do pacote transmitido.                                      |
| `nobattery`     | Formatação    | Remove os bytes finais referentes ao nível de bateria e qualidade dos sensores.                                                   |
| `format-0`      | Formatação    | *(Padrão)* Converte e envia os canais EEG em microvolts com ponto flutuante (ex.: `4201.02564096`).                               |
| `format-1`      | Formatação    | Envia os bytes inteiros brutos descriptografados sem conversão para microvolts.                                                   |
| `format-2`      | Formatação    | Específico para split de bits do Insight em duas metades (8-bit + 6-bit).                                                         |
| `format-3`      | Formatação    | Utilizado para Insight via USB ou firmware específico do EPOC+ via Bluetooth.                                                     |
| `integer`       | Formatação    | Converte os valores de microvoltagem em inteiros sem sinal de 16-bit (*Big Endian*).                                              |
| `float`         | Formatação    | *(Padrão OpenViBE)* Envia os valores de microvoltagem em float de 32-bit (*Big Endian*).                                          |
| `ovdelay:NNN`   | Ajuste Fino   | Define o atraso de envio para o OpenViBE em milissegundos (ex.: `ovdelay:100`). Máximo 999.                                       |
| `ovsamples:NNN` | Ajuste Fino   | Define a quantidade de amostras por bloco no OpenViBE (ex.: `ovsamples:004`). Valores válidos: 004, 016, 032, 064, 128, 256, 512. |
| `allmode`       | Filtro        | Transmite pacotes tanto do EEG quanto do Giroscópio (MEMS).                                                                       |
| `eegmode`       | Filtro        | *(Padrão com `nocounter`)* Transmite unicamente pacotes de EEG, descartando o giroscópio.                                         |
| `gyromode`      | Filtro        | Transmite unicamente pacotes do giroscópio, descartando o EEG.                                                                    |
| `baseline`      | Processamento | Calcula a média móvel contínua do sinal a cada segundo e a transmite (`CyKITv2:::Baseline:::...`).                                |
| `filter`        | Processamento | Quando usado com `baseline`, subtrai o valor do sinal da linha de base (`baseline - sinal`).                                      |
| `blankdata`     | Teste         | Injeta no fluxo um pacote cifrado estático equivalente a sinal EEG neutro (contador fixo em 0).                                   |
| `blankcsv`      | Formatação    | Preenche canais inativos com zeros até totalizar o número total de canais para gravação CSV.                                      |
| `bluetooth`     | Driver        | Ativa o modo Bluetooth Low Energy (BLE) com detecção automática do dispositivo.                                                   |
| `bluetooth=KEY` | Driver        | Conecta a um headset BLE específico através da chave hexadecimal de 8 dígitos do emparelhamento Windows.                          |
| `delimiter=COD` | Formatação    | Altera o separador das colunas informando o código ASCII (padrão é vírgula `,`).                                                  |
| `path`          | Diagnóstico   | Imprime no console a lista de caminhos de módulos Python (`sys.path`).                                                            |

---

### 5. Regras de Agregação de Flags e Exemplos Práticos

#### Regra de Sintaxe

- As flags são unidas exclusivamente pelo caractere **`+`**, sem espaços.
- A ordem das flags é livre.
- Parâmetros com valores numéricos usam dois-pontos `:` (ex.: `ovdelay:100`, `ovsamples:004`) ou igual `=` (ex.:
  `bluetooth=1A2B3C4D`, `delimiter=59`).

#### Exemplos Agregados Práticos (sempre dentro de `CyKIT-master\Py3`)

1. **Diagnóstico completo via USB (usando driver Windows HID):**
   ```powershell
   cd CyKIT-master\Py3
   python.exe .\CyKIT.py 127.0.0.1 54123 6 info+confirm+verbose+outputdata+pywinusb
   ```

2. **Visualização direta no console sem precisar de navegador ou rede:**
   ```powershell
   cd CyKIT-master\Py3
   python.exe .\CyKIT.py 127.0.0.1 54123 6 noweb+outputdata+pywinusb
   ```

3. **Streaming para OpenViBE com buffer otimizado:**
   ```powershell
   cd CyKIT-master\Py3
   python.exe .\CyKIT.py 127.0.0.1 5151 6 openvibe+generic+nocounter+noheader+nobattery+ovdelay:100+float+ovsamples:004+pywinusb
   ```

4. **Transmissão limpa para script próprio via TCP (apenas os 14 canais EEG):**
   ```powershell
   cd CyKIT-master\Py3
   python.exe .\CyKIT.py 127.0.0.1 9000 6 generic+noheader+nocounter+nobattery+eegmode+pywinusb
   ```

5. **Conexão Bluetooth LE com chave de pareamento do Windows:**
   ```powershell
   cd CyKIT-master\Py3
   python.exe .\CyKIT.py 127.0.0.1 54123 6 bluetooth=A1B2C3D4+eegmode+outputdata
   ```