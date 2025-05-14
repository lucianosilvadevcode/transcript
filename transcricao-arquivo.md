## TRANSCRIÇÃO DE ÁUDIO COM ENTRADA POR ARQUIVO
----

1.  **Configuração do Ambiente e Instalações Básicas:**
    *   Python 3.7+
    *   Ambiente virtual
    *   Framework para API (usaremos **FastAPI**)
    *   Servidor ASGI (usaremos **Uvicorn**)
2.  **Escolha e Integração da Biblioteca de Transcrição:**
    *   **Opção A (Mais Simples):** `SpeechRecognition` (usando a API gratuita do Google como backend).
    *   **Opção B (Mais Avançada/Precisa):** `OpenAI Whisper` (rodando localmente).
3.  **Manipulação de Arquivos de Áudio:**
    *   Receber upload de áudio.
    *   Converter formatos de áudio (se necessário, usando `pydub`).
4.  **Criação do Endpoint da API.**
5.  **Execução e Teste.**
6.  **Melhorias Potenciais.**

---

**Passo 0: Pré-requisitos de Sistema**

*   **Python 3.7+:** Verifique sua versão com `python --version` ou `python3 --version`.
*   **pip:** Geralmente vem com o Python.
*   **FFmpeg (Importante!):** A biblioteca `pydub` (que usaremos para conversão de formatos de áudio) depende do `ffmpeg`.
    *   **Linux (Debian/Ubuntu):** `sudo apt update && sudo apt install ffmpeg`
    *   **macOS (usando Homebrew):** `brew install ffmpeg`
    *   **Windows:** Baixe os binários do site oficial do `ffmpeg`, extraia-os e adicione o caminho da pasta `bin` à variável de ambiente `PATH` do sistema.

---

**Passo 1: Configuração do Ambiente Virtual e Instalações Iniciais**

É altamente recomendável usar um ambiente virtual para isolar as dependências do projeto.

```bash
# 1. Crie um diretório para o seu projeto
mkdir minha_api_transcricao
cd minha_api_transcricao

# 2. Crie um ambiente virtual
python -m venv venv_transcricao
# ou python3 -m venv venv_transcricao

# 3. Ative o ambiente virtual
# No Windows:
# venv_transcricao\Scripts\activate
# No Linux/macOS:
source venv_transcricao/bin/activate

# 4. Instale o FastAPI e o Uvicorn
pip install fastapi uvicorn[standard]
```
*   `fastapi`: O framework web que usaremos para construir a API.
*   `uvicorn`: Um servidor ASGI (Asynchronous Server Gateway Interface) para rodar nossa aplicação FastAPI. O `[standard]` instala dependências recomendadas como `python-multipart` para lidar com uploads de formulários.

---

**Passo 2: Escolha e Integração da Biblioteca de Transcrição**

Vamos apresentar duas opções. Comece com a Opção A se quiser algo mais rápido para testar.

**Opção A: Usando `SpeechRecognition` (com Google Web Speech API)**

Esta biblioteca é um wrapper para várias APIs de reconhecimento de fala. Usaremos o Google Web Speech API, que é gratuito para uso limitado e não requer chave de API, mas não é ideal para produção em larga escala ou dados sensíveis.

```bash
# 5. Instale SpeechRecognition e pydub (dentro do ambiente virtual ativado)
pip install SpeechRecognition pydub
```
*   `SpeechRecognition`: Biblioteca para realizar o reconhecimento de fala.
*   `pydub`: Para manipular arquivos de áudio (ler diferentes formatos e converter para WAV, que é bem suportado pelo `SpeechRecognition`).

**Opção B: Usando `OpenAI Whisper` (Localmente)**

Whisper é um modelo de transcrição de última geração, muito preciso. Pode ser executado localmente, mas requer mais recursos (CPU/GPU) e o download do modelo.

```bash
# 5. Instale OpenAI Whisper (dentro do ambiente virtual ativado)
pip install openai-whisper
# Para melhor performance com GPU NVIDIA (se tiver e CUDA configurado):
# pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
# (Ajuste 'cu118' para sua versão do CUDA Toolkit. Whisper tentará usar GPU se o PyTorch com CUDA estiver disponível)
```

---

**Passo 3: Criar o Arquivo Principal da API (ex: `main.py`)**

Crie um arquivo chamado `main.py` no diretório `minha_api_transcricao`.
Vamos começar com a estrutura básica e depois adicionar a lógica de transcrição.

```python
# main.py
from fastapi import FastAPI, File, UploadFile, HTTPException
import uvicorn # Para a parte de execução, se não for rodar pelo terminal

# --- Importações específicas da biblioteca de transcrição (adicionar conforme a opção escolhida) ---
# Para Opção A (SpeechRecognition):
import speech_recognition as sr
from pydub import AudioSegment
import io # Para manipular bytes como arquivos
import os # Para trabalhar com extensões de arquivo

# Para Opção B (Whisper):
# import whisper
# import tempfile # Para salvar o áudio temporariamente para o Whisper

# -----------------------------------------------------------------------------------------------

app = FastAPI(
    title="API de Transcrição de Áudio",
    description="Uma API para transcrever áudios enviados.",
    version="1.0.0"
)

# --- Carregamento do modelo (para Opção B - Whisper) ---
# MODELO_WHISPER = None # Inicializar como None
# try:
#     # Escolha o modelo: "tiny", "base", "small", "medium", "large"
#     # Modelos menores são mais rápidos, maiores são mais precisos.
#     # O modelo será baixado na primeira vez.
#     MODELO_WHISPER = whisper.load_model("base")
#     print(f"Modelo Whisper 'base' carregado no dispositivo: {MODELO_WHISPER.device}")
# except Exception as e:
#     print(f"Erro ao carregar o modelo Whisper: {e}. A funcionalidade de transcrição com Whisper pode não funcionar.")
# --------------------------------------------------------

@app.get("/")
async def root():
    return {"message": "Bem-vindo à API de Transcrição de Áudio! Use /docs para ver os endpoints."}

# --- AQUI VAI O ENDPOINT DE TRANSCRIÇÃO ---

# if __name__ == "__main__":
# uvicorn.run(app, host="0.0.0.0", port=8000)
# É mais comum rodar uvicorn main:app --reload pelo terminal
```

---

**Passo 4: Criar o Endpoint de Transcrição**

Adicione o seguinte código ao seu `main.py`, substituindo o comentário `# --- AQUI VAI O ENDPOINT DE TRANSCRIÇÃO ---`.

**Para Opção A: `SpeechRecognition`**

```python
# main.py (continuando...)

recognizer = sr.Recognizer() # Inicializa o reconhecedor (apenas para Opção A)

@app.post("/transcrever/speech_recognition", tags=["Transcrição"])
async def transcrever_audio_sr(arquivo_audio: UploadFile = File(...)):
    """
    Recebe um arquivo de áudio (WAV, MP3, OGG, etc.),
    converte para WAV se necessário, e transcreve usando Google Web Speech API.
    """
    nome_arquivo = arquivo_audio.filename
    conteudo_arquivo = await arquivo_audio.read() # Ler o conteúdo do arquivo em bytes
    stream_audio = io.BytesIO(conteudo_arquivo)

    try:
        # Tenta identificar o formato pela extensão
        _, extensao = os.path.splitext(nome_arquivo.lower())

        if extensao == ".wav":
            som_audio = AudioSegment.from_wav(stream_audio)
        elif extensao == ".mp3":
            som_audio = AudioSegment.from_mp3(stream_audio)
        elif extensao == ".ogg":
            som_audio = AudioSegment.from_ogg(stream_audio)
        # Adicione mais formatos conforme necessário (ex: .flac)
        # elif extensao == ".flac":
        #     som_audio = AudioSegment.from_file(stream_audio, format="flac")
        else:
            # Tentar detectar o formato automaticamente (pode falhar para alguns)
            try:
                som_audio = AudioSegment.from_file(stream_audio)
            except Exception as e_format:
                raise HTTPException(
                    status_code=400,
                    detail=f"Formato de áudio '{extensao}' não suportado ou arquivo inválido. Detalhe: {e_format}. Tente WAV, MP3 ou OGG."
                )

        # Exportar o áudio para o formato WAV em memória (SpeechRecognition lida melhor com WAV)
        wav_em_memoria = io.BytesIO()
        som_audio.export(wav_em_memoria, format="wav")
        wav_em_memoria.seek(0) # Voltar ao início do stream para leitura

        # Usar o arquivo WAV em memória com SpeechRecognition
        with sr.AudioFile(wav_em_memoria) as source:
            dados_audio = recognizer.record(source)  # Ler o arquivo de áudio completo

        # Transcrever usando Google Web Speech API (padrão)
        # Especifique o idioma para melhor precisão, ex: "pt-BR" para Português do Brasil
        texto_transcrito = recognizer.recognize_google(dados_audio, language="pt-BR")
        return {"nome_arquivo": nome_arquivo, "transcricao": texto_transcrito}

    except sr.UnknownValueError:
        raise HTTPException(status_code=400, detail="O Google Speech Recognition não conseguiu entender o áudio.")
    except sr.RequestError as e:
        raise HTTPException(status_code=503, detail=f"Não foi possível solicitar resultados do serviço Google Speech Recognition; {e}")
    except Exception as e:
        # Captura outras exceções (ex: erro de pydub ao carregar áudio)
        raise HTTPException(status_code=500, detail=f"Erro ao processar o áudio: {str(e)}")
```

**Para Opção B: `OpenAI Whisper`** (Descomente as importações e o carregamento do modelo no Passo 3)

```python
# main.py (continuando...)
# Certifique-se que as importações 'import whisper' e 'import tempfile' e o
# carregamento do MODELO_WHISPER estejam descomentados no início do arquivo.

@app.post("/transcrever/whisper", tags=["Transcrição"])
async def transcrever_audio_whisper(arquivo_audio: UploadFile = File(...)):
    """
    Recebe um arquivo de áudio, salva temporariamente e transcreve usando OpenAI Whisper.
    """
    if MODELO_WHISPER is None:
        raise HTTPException(status_code=503, detail="Modelo Whisper não está carregado ou falhou ao carregar.")

    conteudo_arquivo = await arquivo_audio.read()

    # Whisper geralmente precisa de um caminho de arquivo, então salvamos temporariamente.
    # Usamos um arquivo temporário nomeado para garantir que Whisper possa acessá-lo
    # e que tenha a extensão correta para o Whisper identificar o formato.
    try:
        sufixo_arquivo = os.path.splitext(arquivo_audio.filename)[1]
        with tempfile.NamedTemporaryFile(delete=False, suffix=sufixo_arquivo) as tmp_audio:
            tmp_audio.write(conteudo_arquivo)
            caminho_audio_temporario = tmp_audio.name

        # Transcrever o áudio
        # O Whisper detecta o idioma automaticamente, mas pode ser especificado.
        # fp16=False é recomendado para CPU. Se tiver GPU com suporte, pode ser True.
        resultado = MODELO_WHISPER.transcribe(caminho_audio_temporario, language="pt", fp16=False)
        texto_transcrito = resultado["text"]

        return {
            "nome_arquivo": arquivo_audio.filename,
            "transcricao": texto_transcrito,
            "idioma_detectado": resultado.get("language", "N/A")
        }

    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Erro durante a transcrição com Whisper: {str(e)}")
    finally:
        # Limpar o arquivo temporário
        if 'caminho_audio_temporario' in locals() and os.path.exists(caminho_audio_temporario):
            os.remove(caminho_audio_temporario)
```

---

**Passo 5: Executar a API**

No seu terminal, dentro do diretório `minha_api_transcricao` e com o ambiente virtual `venv_transcricao` ativado:

```bash
uvicorn main:app --reload
```
*   `main`: o nome do seu arquivo Python (sem o `.py`).
*   `app`: o nome da instância FastAPI que você criou (`app = FastAPI()`).
*   `--reload`: faz o servidor reiniciar automaticamente quando você salva alterações no código (ótimo para desenvolvimento).

Você deverá ver uma saída como:
`INFO: Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)`

---

**Passo 6: Testar a API**

1.  **Interface Swagger UI (Recomendado):**
    Abra seu navegador e vá para `http://127.0.0.1:8000/docs`.
    O FastAPI gera automaticamente uma documentação interativa da API.
    *   Você verá o endpoint que criou (`/transcrever/speech_recognition` ou `/transcrever/whisper`).
    *   Expanda-o, clique em "Try it out".
    *   Em "arquivo\_audio", clique em "Choose File" e selecione um arquivo de áudio do seu computador (ex: `.wav`, `.mp3`, `.ogg`).
    *   Clique em "Execute".

2.  **Usando `curl` (Linha de Comando):**
    Se você tiver um arquivo de áudio chamado `teste.mp3` no mesmo diretório do terminal:

    Para `SpeechRecognition`:
    ```bash
    curl -X POST -F "arquivo_audio=@teste.mp3" http://127.0.0.1:8000/transcrever/speech_recognition
    ```
    Para `Whisper`:
    ```bash
    curl -X POST -F "arquivo_audio=@teste.mp3" http://127.0.0.1:8000/transcrever/whisper
    ```

Você deverá receber uma resposta JSON com a transcrição.

---

**Passo 7: Melhorias Potenciais (Próximos Passos)**

1.  **Processamento Assíncrono para Áudios Longos:** Para arquivos de áudio grandes, a transcrição pode demorar. A requisição HTTP ficaria bloqueada.
    *   **FastAPI Background Tasks:** Para tarefas que não precisam retornar o resultado imediatamente na mesma requisição.
    *   **Sistemas de Fila (Celery + Redis/RabbitMQ):** Solução mais robusta para processamento assíncrono e distribuído. O cliente enviaria o áudio, receberia um ID de tarefa, e depois consultaria o status/resultado dessa tarefa.
2.  **Validação e Limites:**
    *   Limitar o tamanho do arquivo de upload.
    *   Validar tipos de arquivo permitidos de forma mais estrita.
3.  **Autenticação e Autorização:** Proteger sua API se ela for exposta publicamente (ex: API Keys, OAuth2).
4.  **Logging Detalhado:** Registrar informações sobre requisições, sucessos e falhas.
5.  **Configuração Avançada do Modelo (Whisper):** Permitir que o usuário escolha o tamanho do modelo Whisper via parâmetro ou configuração, ou especificar mais opções de transcrição (`temperature`, `beam_size`, etc.).
6.  **Dockerização:** Empacotar sua API em um container Docker para facilitar o deploy em diferentes ambientes.
7.  **Testes Unitários e de Integração:** Escrever testes para garantir que sua API funcione como esperado.
8.  **Escalabilidade:** Se você espera muitas requisições, precisará pensar em como escalar sua API (ex: múltiplos workers Uvicorn, Kubernetes).

---
