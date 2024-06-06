# Documentação do Projeto: Sistema de Identificação e Contagem de Garrafas

---

## Índice
1. [Introdução](#introdução)
2. [Estrutura do Projeto](#estrutura-do-projeto)
3. [Configuração do Ambiente](#configuração-do-ambiente)
4. [Configuração do Banco de Dados](#configuração-do-banco-de-dados)
5. [Configuração do Servidor Flask](#configuração-do-servidor-flask)
6. [Código da ESP32-CAM](#código-da-esp32-cam)
7. [Execução do Projeto](#execução-do-projeto)
8. [Rotas e Funcionalidades](#rotas-e-funcionalidades)
9. [Testes e Verificação](#testes-e-verificação)

---

## Introdução

O projeto consiste em um sistema automatizado de identificação e contagem de diferentes tipos de garrafas utilizando uma ESP32-CAM e uma interface web construída com Flask. O sistema identifica os tipos de garrafas, contabiliza-as e armazena os dados em um banco de dados MySQL, permitindo a visualização dos dados através de uma página web.

---

## Estrutura do Projeto

```
pi03-05/
│
├── .env
├── .gitignore
├── db.py
├── server.py
├── requirements.txt
├── static/
│   └── css/
│       └── styles.css
├── templates/
│   ├── about.html
│   ├── index.html
│   └── steps.html
├── utils.py
└── espcam_detection.py
```


## Configuração do Ambiente

### 1. Criar e Ativar o Ambiente Virtual

```sh
python -m venv venv
source venv/bin/activate  # Para Linux/Mac
.\venv\Scripts\activate   # Para Windows
```

### 2. Instalar Dependências

```sh
pip install -r requirements.txt
```

### 3. Arquivo `.env`

Crie um arquivo `.env` na raiz do projeto e adicione as variáveis de ambiente:

```
DB_USER=seu_usuario
DB_PASSWORD=sua_senha
DB_HOST=localhost
DB_NAME=nome_do_banco_de_dados
```

---

## Configuração do Banco de Dados

### 1. Criar o Banco de Dados e a Tabela

```sql
CREATE DATABASE nome_do_banco_de_dados;

USE nome_do_banco_de_dados;

CREATE TABLE Garrafas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    tipo VARCHAR(255) NOT NULL,
    qtd INT NOT NULL,
    preco DECIMAL(10, 2) NOT NULL,
    quantidade_ml INT NOT NULL
);
```

---

## Configuração do Servidor Flask

### 1. Código do `server.py`

```python
from flask import Flask, render_template, request, jsonify
import logging
from db import get_db_connection

app = Flask(__name__)

logging.basicConfig(level=logging.INFO)

# Definir preços e quantidades fixas
PRECOS_FIXOS = {
    "Guarana": 1.50,
    "Pepsi": 2.00
}

QUANTIDADE_FIXA = {
    "Guarana": 350,
    "Pepsi": 500
}

def enviar_dados(tipo_garrafa):
    cnx = get_db_connection()
    if cnx is None:
        logging.error("Failed to connect to the database.")
        return False

    try:
        cursor = cnx.cursor()
        check_query = "SELECT qtd FROM Garrafas WHERE tipo = %s"
        cursor.execute(check_query, (tipo_garrafa,))
        result = cursor.fetchone()

        preco = PRECOS_FIXOS.get(tipo_garrafa, 0)
        quantidade_ml = QUANTIDADE_FIXA.get(tipo_garrafa, 0)

        if result:
            update_query = "UPDATE Garrafas SET qtd = qtd + 1, preco = %s, quantidade_ml = %s WHERE tipo = %s"
            cursor.execute(update_query, (preco, quantidade_ml, tipo_garrafa))
        else:
            insert_query = "INSERT INTO Garrafas (tipo, qtd, preco, quantidade_ml) VALUES (%s, 1, %s, %s)"
            cursor.execute(insert_query, (tipo_garrafa, preco, quantidade_ml))
        
        cnx.commit()
        cursor.close()
        logging.info(f"Successfully updated or inserted {tipo_garrafa} count in the database.")
        return True
    except Exception as e:
        logging.error(f"Error updating data: {e}")
        return False
    finally:
        cnx.close()

@app.route('/receber_dados', methods=['POST'])
def receber_dados():
    dados = request.get_json()
    if dados and 'tipo_garrafa' in dados:
        logging.info(f"Recebendo dados: {dados['tipo_garrafa']}")
        if enviar_dados(dados['tipo_garrafa']):
            logging.info("Dados processados com sucesso!")
            return 'Dados recebidos com sucesso!', 200
        else:
            logging.error("Erro ao processar os dados.")
            return 'Erro ao processar os dados.', 500
    else:
        logging.error("Dados inválidos recebidos.")
        return 'Dados inválidos.', 400

@app.route('/')
@app.route('/home')
def home():
    cnx = get_db_connection()
    if cnx is None:
        logging.error("Failed to connect to the database.")
        return render_template('index.html', dados=[])

    try:
        cursor = cnx.cursor()
        query = "SELECT tipo, qtd, preco, quantidade_ml FROM Garrafas"
        cursor.execute(query)
        dados = cursor.fetchall()
        cursor.close()
        return render_template('index.html', dados=dados)
    except Exception as e:
        logging.error(f"Error fetching data: {e}")
        return render_template('index.html', dados=[])
    finally:
        cnx.close()

@app.route('/about')
def about_page():
    return render_template('about.html')

@app.route('/steps')
def steps_page():
    return render_template('steps.html')

@app.route('/consulta_dados')
def consulta_dados():
    cnx = get_db_connection()
    if cnx is None:
        logging.error("Failed to connect to the database.")
        return 'Failed to connect to the database.', 500

    try:
        cursor = cnx.cursor()
        query = "SELECT tipo, qtd, preco, quantidade_ml FROM Garrafas"
        cursor.execute(query)
        dados = cursor.fetchall()
        cursor.close()
        logging.info("Consulta de dados bem-sucedida.")
        return jsonify(dados), 200
    except Exception as e:
        logging.error(f"Error during data query: {e}")
        return f"Error during data query: {e}", 500
    finally:
        cnx.close()

if __name__ == '__main__':
    app.run(debug=True)
```

### 2. Código do `db.py`

```python
import mysql.connector
import os
from mysql.connector import Error
from dotenv import load_dotenv
import logging

# Carregar as variáveis de ambiente do arquivo .env
load_dotenv()

# Configuração do logging
logging.basicConfig(level=logging.INFO)

def get_db_connection():
    """
    Função para obter a conexão com o banco de dados MySQL.
    Retorna a conexão se bem-sucedida, caso contrário, None.
    """
    try:
        connection = mysql.connector.connect(
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            host=os.getenv('DB_HOST'),
            database=os.getenv('DB_NAME')
        )
        if connection.is_connected():
            logging.info("Successfully connected to the database.")
            return connection
    except Error as e:
        logging.error(f"Error while connecting to MySQL: {e}")
        return None

def close_db_connection(connection):
    """
    Função para fechar a conexão com o banco de dados.
    """
    if connection.is_connected():
        connection.close()
        logging.info("The database connection is closed.")
```

---

## Código da ESP32-CAM

### 1. Código do `espcam_detection.py`

```python
import argparse
import sys
import time
import urllib.request
import cv2
import mediapipe as mp
import numpy as np
import requests
from mediapipe.tasks import python
from mediapipe.tasks.python import vision

from utils import visualize

# Variáveis globais para calcular FPS
CONTADOR, FPS = 0, 0
TEMPO_INICIAL = time.time()
url = 'http://192.168.18.105/800x600.jpg'  # URL da câmera

# Variáveis globais para contar detecções
contagem_produtos = {"Guarana": 0, "Pepsi": 0}

# Dicionário para rastrear o último momento em que cada produto foi detectado
last_seen = {}

def rodar(modelo: str, max_resultados: int, limiar_pontuacao: float,
          id_camera: int, largura: int, altura: int) -> None:
    """Executa continuamente a inferência nas imagens adquiridas da câmera.

    Args:
        modelo: Nome do modelo de detecção de objetos TFLite.
        max_resultados: Número máximo de resultados de detecção.
        limiar_pontuacao: O limiar de pontuação dos resultados da detecção.
        id_camera:

 O ID da câmera a ser passado para o OpenCV.
        largura: A largura do quadro capturado pela câmera.
        altura: A altura do quadro capturado pela câmera.
    """

    # Parâmetros de visualização
    tamanho_linha = 50  # pixels
    margem_esquerda = 24  # pixels
    cor_texto = (0, 0, 0)  # preto
    tamanho_fonte = 1
    espessura_fonte = 1
    contagem_frames_fps_avg = 10

    quadro_deteccao = None
    lista_resultados_deteccao = []

    def salvar_resultado(resultado: vision.ObjectDetectorResult, imagem_saida: mp.Image, timestamp_ms: int):
        global FPS, CONTADOR, TEMPO_INICIAL, contagem_produtos, last_seen

        # Calcular o FPS
        if CONTADOR % contagem_frames_fps_avg == 0:
            FPS = contagem_frames_fps_avg / (time.time() - TEMPO_INICIAL)
            TEMPO_INICIAL = time.time()

        current_time = time.time()

        for deteccao in resultado.detections:
            nome_categoria = deteccao.categories[0].category_name

            if nome_categoria not in last_seen or (current_time - last_seen[nome_categoria]) > 1:
                if nome_categoria == "GUARANA":
                    contagem_produtos["Guarana"] += 1
                    print("Guarana detectado! Total de Guarana:", contagem_produtos["Guarana"])
                    enviar_dados_para_servidor("Guarana")  # Enviar dados ao servidor Flask
                elif nome_categoria == "Pepsi":
                    contagem_produtos["Pepsi"] += 1
                    print("Pepsi detectado! Total de Pepsi:", contagem_produtos["Pepsi"])
                    enviar_dados_para_servidor("Pepsi")  # Enviar dados ao servidor Flask

                last_seen[nome_categoria] = current_time

        # Atualizar tempo de última visualização para as detecções atuais
        for nome_categoria in last_seen.keys():
            if nome_categoria not in [d.categories[0].category_name for d in resultado.detections]:
                last_seen[nome_categoria] = current_time

        lista_resultados_deteccao.append(resultado)
        CONTADOR += 1

    # Função para enviar dados ao servidor Flask
    def enviar_dados_para_servidor(tipo_garrafa):
        try:
            # Altere o URL abaixo para o endereço do seu servidor Flask
            response = requests.post('http://127.0.0.1:5000/receber_dados', json={'tipo_garrafa': tipo_garrafa})
            if response.status_code == 200:
                print(f'Dados de {tipo_garrafa} enviados com sucesso!')
            else:
                print(f'Falha ao enviar dados de {tipo_garrafa}. Código de status: {response.status_code}')
        except Exception as e:
            print(f'Erro ao enviar dados para o servidor: {e}')

    # Inicializar o modelo de detecção de objetos
    opcoes_base = python.BaseOptions(model_asset_path=modelo)
    opcoes = vision.ObjectDetectorOptions(base_options=opcoes_base,
                                          running_mode=vision.RunningMode.LIVE_STREAM,
                                          max_results=max_resultados, score_threshold=limiar_pontuacao,
                                          result_callback=salvar_resultado)
    detector = vision.ObjectDetector.create_from_options(opcoes)

    # Capturar continuamente imagens da câmera e executar a inferência
    while True:
        try:
            imgResp = urllib.request.urlopen(url)
            imgNp = np.array(bytearray(imgResp.read()), dtype=np.uint8)
            imagem = cv2.imdecode(imgNp, 1)
        except Exception as e:
            print(f'Erro ao acessar a câmera: {e}')
            time.sleep(5)  # Esperar antes de tentar novamente
            continue

        # Converter a imagem de BGR para RGB conforme necessário pelo modelo TFLite.
        imagem_rgb = cv2.cvtColor(imagem, cv2.COLOR_BGR2RGB)
        mp_imagem = mp.Image(image_format=mp.ImageFormat.SRGB, data=imagem_rgb)

        # Executar detecção de objetos usando o modelo.
        detector.detect_async(mp_imagem, time.time_ns() // 1_000_000)

        # Mostrar o FPS
        texto_fps = 'FPS = {:.1f}'.format(FPS)
        localizacao_texto = (margem_esquerda, tamanho_linha)
        quadro_atual = imagem
        cv2.putText(quadro_atual, texto_fps, localizacao_texto, cv2.FONT_HERSHEY_DUPLEX,
                    tamanho_fonte, cor_texto, espessura_fonte, cv2.LINE_AA)

        # Exibir contagem de produtos
        texto_contagem = f"Guarana: {contagem_produtos['Guarana']} Pepsi: {contagem_produtos['Pepsi']}"
        localizacao_contagem = (margem_esquerda, tamanho_linha * 2)
        cv2.putText(quadro_atual, texto_contagem, localizacao_contagem, cv2.FONT_HERSHEY_DUPLEX,
                    tamanho_fonte, cor_texto, espessura_fonte, cv2.LINE_AA)

        if lista_resultados_deteccao:
            quadro_atual = visualize(quadro_atual, lista_resultados_deteccao[0])
            quadro_deteccao = quadro_atual
            lista_resultados_deteccao.clear()

        if quadro_deteccao is not None:
            cv2.imshow('detecao_de_objetos', quadro_deteccao)

        # Parar o programa se a tecla ESC for pressionada.
        if cv2.waitKey(1) == 27:
            break

    detector.close()
    cv2.destroyAllWindows()

def main():
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter)
    parser.add_argument(
        '--modelo',
        help='Caminho do modelo de detecção de objetos.',
        required=False,
        default='GUARANA.tflite')
    parser.add_argument(
        '--maxResultados',
        help='Número máximo de resultados de detecção.',
        required=False,
        default=5)
    parser.add_argument(
        '--limiarPontuacao',
        help='O limiar de pontuação dos resultados da detecção.',
        required=False,
        type=float,
        default=0.80)
    parser.add_argument(
        '--idCamera', help='Id da câmera.', required=False, type=int, default=0)
    parser.add_argument(
        '--larguraQuadro',
        help='Largura do quadro a capturar da câmera.',
        required=False,
        type=int, default=640)
    parser.add_argument(
        '--alturaQuadro',
        help='Altura do quadro a capturar da câmera.',
        required=False, type=int, default=480)
    args = parser.parse_args()

    rodar(args.modelo, int(args.maxResultados),
          args.limiarPontuacao, int(args.idCamera), args.larguraQuadro, args.alturaQuadro)

if __name__ == '__main__':
    main()
```

---

## Execução do Projeto

### 1. Iniciar o Servidor Flask
- Navegue até o diretório do projeto e execute:
  ```sh
  python server.py
  ```

### 2. Executar o Código da ESP32-CAM
- Em um novo terminal, execute:
  ```sh
  python espcam_detection.py --modelo GUARANA.tflite --maxResultados 5 --limiarPontuacao 0.80 --idCamera 0 --larguraQuadro 640 --alturaQuadro 480
  ```

### 3. Consultar os Dados
- Abra um navegador e acesse:
  ```plaintext
  http://127.0.0.1:5000/consulta_dados
  ```

---

## Rotas e Funcionalidades

### `server.py`

- **`/receber_dados` (POST)**:
  Recebe os dados do tipo de garrafa e atualiza o banco de dados.

- **`/home` ou `/` (GET)**:
  Renderiza a página principal com os dados da tabela `Garrafas`.

- **`/about` (GET)**:
  Renderiza a página "Sobre Nós".

- **`/steps` (GET)**:
  Renderiza a página "Passo a Passo".

- **`/consulta_dados` (GET)**:
  Retorna os dados da tabela `Garrafas` em formato JSON.

---

## Testes e Verificação

### 1. Testar Inserção e Consulta
- Utilize as rotas `/test_insert` e `/test_select` para verificar a funcionalidade de inserção e consulta.

### 2. Monitorar Logs
- Verifique os logs no terminal para mensagens de sucesso ou erro.

### 3. Verificar a Página Principal
- Acesse a página principal para visualizar os dados atualizados.

---

## Conclusão

Este sistema integra uma ESP32-CAM para identificar e contar tipos de garrafas, armazenando os dados em um banco de dados MySQL e apresentando-os em uma interface web construída com Flask. A documentação fornece uma visão geral do projeto, estrutura, configuração, execução e rotas disponíveis para interação.
```
