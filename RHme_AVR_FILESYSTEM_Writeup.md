# RHMe - AVR FILE SYSTEM Write-up

Bem-vindo(a) ao nosso primeiro write-up dos desafios RHMe! :D

## 1 - Arquivo HEX

Em primeiro lugar, devemos inserir o arquivo `avr_filesystem.hex` no Arduino. Como não conseguimos inserir esse arquivo da mesma forma que um arquivo `.ino`, devemos executar este comando no PowerShell/CLI:

```bash
avrdude -c arduino -p atmega328p -P /dev/ttyUSB* -b115200 -u -V -U flash:w:<filename>
```

> Substitua o `*` pela porta serial usada (caso não saiba, rode `ls /dev/ttyUSB* /dev/ttyACM*` no terminal para verificar). Substitua `<filename>` pelo nome do arquivo que você deseja enviar para o Arduino.

Após isso, o arquivo deve ter sido inserido com sucesso no firmware do Arduino.

Abra a IDE do Arduino, selecione seu Arduino e entre no **Serial Monitor**. Você verá este output:

```plaintext
RHMeOS file API
Files in system:

drwxrwxr-x remote remote 4096 sep  1 .
drwxrwxr-x remote remote 4096 sep  1 ..
-r--r--r-- remote remote   87 sep 14 cat.txt
-r--r--r-- remote remote   47 sep 16 finances.csv
-r--r--r-- remote remote    3 sep 14 joke.txt
-rw------- root   root     37 jan  5 passwd
-rw------- root   root      8 jan  1 pepper

Request?
>>
```

---

## 2 - Início dos Testes

Primeiramente, testamos o exemplo listado no README:

```plaintext
933d86ae930c9a5d6d3a334297d9e72852f05c57#cat.txt:finances.csv
```

Ao inseri-lo no Serial Monitor, temos este retorno:

```plaintext
cat.txt:
 A_A
 (-.-)
  | |
 /   \
|     |   __
|  || |  |  \__
 \_||_/_/

finances.csv:
year,profit
2014,+100%%
2015,+200%%
```

Isso significa que o Arduino está recebendo um **token via Serial Monitor** e, com base nesse token, ele retorna arquivos armazenados no sistema de arquivos interno. Se o firmware estiver verificando esse token com um **hash inseguro**, ele pode ser vulnerável a um **ataque de extensão de comprimento**.

Se tentarmos acessar outros arquivos com o mesmo token inicial, o Arduino simplesmente nos ignora, sem enviar mensagens de erro:

```plaintext
Request?
933d86ae930c9a5d6d3a334297d9e72852f05c57#cat.txt:passwd

Request?
>>
```

---

## 3 - Ataque de Extensão de Comprimento

Para explicar de forma simples, um **ataque de extensão de comprimento de hash** funciona assim:

Imagine que você tem um **cofre mágico** que só abre se você disser a **palavra secreta correta**. Mas, em vez de verificar a palavra inteira, o cofre apenas olha para um **código mágico gerado a partir dela**.

Agora, imagine que um ladrão descobre esse **código mágico** para uma palavra secreta que já abre o cofre. Normalmente, ele não saberia mais nada além disso... **mas esse cofre tem um problema!**

Se o ladrão adicionar **mais palavras no final**, ele ainda pode criar um **novo código mágico válido** e abrir o cofre! Ele **não precisa saber a palavra secreta inteira**, basta usar a que já funciona e acrescentar algo no final.

Trazendo isso para o nosso desafio:
- **O Arduino é o cofre**.
- **O token é a palavra secreta**.
- **O hash (código mágico) é o que o Arduino usa para verificar o token**.
- **O hacker pega um token válido e consegue adicionar mais dados sem o Arduino perceber**.

Para automatizar esse ataque, usamos o **hashpumpy**, uma ferramenta que pode ser integrada ao Python.

---

## 4 - Mão na Massa >:)

Criamos um código em Python para explorar essa falha:

```python
import serial
import argparse
import signal
import time
import sys
import hashpumpy

# Dicionário de hashes
hashes = {
    'cat.txt': "96103df3b928d9edc5a103d690639c94628824f5",
    'joke.txt': "715b21027dca61235e2663e59a9bdfb387ca7997",
    'finances.csv': "0b939251f4c781f43efef804ee8faec0212f1144",
    'cat.txt:finances.csv': "933d86ae930c9a5d6d3a334297d9e72852f05c57"
}

EXPECTED_REQUEST_STRING = ">>"

def build_payload(hash_value, payload):
    return f"{hash_value}#{payload}\r\n".encode('utf-8')

def wait_for_prompt(ser):
    buffer = ""
    while True:
        if ser.in_waiting > 0:
            try:
                line = ser.readline().decode('utf-8').strip()
                buffer += line + "\n"
                if "Request?" in line or ">>" in line:
                    print(buffer)
                    return
            except Exception as e:
                print(f"Erro na leitura serial: {e}")

def signal_handler(signal, frame):
    print("[+] Encerrando...")
    ser.close()
    sys.exit(0)

parser = argparse.ArgumentParser(description='Exploit para o desafio RHMe2.')
parser.add_argument('port', help='Porta serial do Arduino')
args = parser.parse_args()

signal.signal(signal.SIGINT, signal_handler)
ser = serial.Serial(args.port, baudrate=19200, bytesize=serial.EIGHTBITS, parity=serial.PARITY_NONE, timeout=2)

ser.open()
wait_for_prompt(ser)

# Testando exploração
target_file = 'cat.txt'
for i in range(20):
    new_hash, new_payload = hashpumpy.hashpump(hashes[target_file], target_file, ':passwd', i)
    print(f"[i = {i}] Enviando: {new_payload}")
    ser.write(build_payload(new_hash, new_payload))
    wait_for_prompt(ser)

ser.close()
sys.exit(0)
```

Esse código testa diferentes variações do hash e envia os requests para o Arduino automaticamente. Se o ataque for bem-sucedido, ele retorna:

```plaintext
Flag encontrada!! 
```