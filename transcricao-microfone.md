## TRANSCRIÇÃO DE ÁUDIO COM ENTRADA PELO MICROFONE
---

A captura de áudio do microfone deve acontecer no **lado do cliente** (ex: em JavaScript no navegador, ou em um aplicativo Python desktop/móvel). Depois de capturado, o áudio pode ser enviado para a nossa API Python para transcrição.

Existem duas abordagens principais para isso:

1.  **Gravar e Enviar (Chunking):** O cliente grava um trecho de áudio (ex: 5-10 segundos), envia esse trecho para a API, recebe a transcrição, e repete. É mais simples de implementar.
2.  **Streaming (WebSockets):** O cliente envia um fluxo contínuo de dados de áudio para a API via WebSockets. A API processa esses pedaços em tempo real (ou quase). É mais complexo, mas oferece uma experiência mais "ao vivo".

**Vamos focar na Abordagem 1 (Gravar e Enviar), pois ela requer menos modificações na API backend que já construímos.** A API backend continuará esperando um arquivo de áudio (ou dados que se pareçam com um arquivo). A diferença é que esse "arquivo" será gerado dinamicamente pelo cliente a partir da entrada do microfone.

**O que NÃO muda (ou muda pouco) na API Python Backend:**

*   A lógica de transcrição com `SpeechRecognition` ou `Whisper` permanece a mesma, pois elas operam sobre dados de áudio, não importa a origem original.
*   O endpoint da API ainda esperará um `UploadFile` ou dados de áudio em um formato conhecido (WAV, MP3, etc.).

**O que PRECISA ser feito (Principalmente no Lado do Cliente):**

1.  **No Cliente (ex: Navegador com JavaScript):**
    *   Solicitar permissão para usar o microfone.
    *   Usar APIs do navegador (como `MediaRecorder API`) para capturar o áudio.
    *   Codificar o áudio capturado em um formato suportado pela API (ex: WAV, Opus, ou o formato que o `MediaRecorder` gerar).
    *   Enviar esses dados de áudio (como um `Blob` ou `File` objeto) para o endpoint da API FastAPI.

**Modificações na API Python (Mínimas, mais contextuais):**

A API em si não precisa de grandes alterações estruturais se o cliente enviar os dados do microfone como se fosse um arquivo. O principal é entender que a "fonte" desse arquivo agora é uma gravação em tempo real do cliente.

Vamos assumir que o cliente irá gravar um pedaço de áudio e enviá-lo como um arquivo (por exemplo, em formato WAV). O endpoint que já criamos para `UploadFile` funcionaria.

**Passo a Passo (com foco na interação cliente-servidor):**

**1. API Backend (Python FastAPI - Praticamente a mesma do tutorial anterior)**

Usaremos a versão com `SpeechRecognition` para simplicidade, mas a lógica para `Whisper` seria similar (receber o áudio, salvar temporariamente, transcrever).

```python
# main.py (API Backend)
from fastapi import FastAPI, File, UploadFile, HTTPException
import speech_recognition as sr
from pydub import AudioSegment
import io
import os
import uvicorn

app = FastAPI(
    title="API de Transcrição de Áudio (para entrada de Microfone via Cliente)",
    description="Recebe áudio (gravado pelo cliente) e transcreve.",
    version="1.1.0"
)

recognizer = sr.Recognizer()

@app.post("/transcrever_microfone/", tags=["Transcrição de Microfone"])
async def transcrever_audio_capturado(arquivo_audio: UploadFile = File(...)):
    """
    Recebe um arquivo de áudio (idealmente WAV, capturado pelo cliente a partir do microfone)
    e tenta transcrevê-lo.
    """
    nome_arquivo = arquivo_audio.filename
    conteudo_arquivo = await arquivo_audio.read()
    stream_audio = io.BytesIO(conteudo_arquivo)

    print(f"Recebido arquivo '{nome_arquivo}' com {len(conteudo_arquivo)} bytes.")

    try:
        # Idealmente, o cliente já envia em WAV.
        # Se o cliente puder enviar em outros formatos, a lógica de conversão original se aplica.
        # Para simplificar, vamos assumir que o cliente envia WAV ou um formato que pydub possa ler diretamente.
        try:
            # Tenta carregar diretamente, pydub é bom em detectar formatos
            som_audio = AudioSegment.from_file(stream_audio)
            print("Áudio carregado com pydub.")
        except Exception as e_pydub:
            # Se falhar, pode ser que o formato não seja diretamente suportado ou precise de uma extensão.
            # Se o cliente sempre enviar WAV, a linha abaixo pode ser mais direta:
            # stream_audio.seek(0) # Resetar o ponteiro do stream
            # som_audio = AudioSegment.from_wav(stream_audio)
            print(f"Erro ao carregar áudio com pydub: {e_pydub}. Tentando tratar como WAV bruto se possível.")
            # Para este exemplo, vamos prosseguir, mas em produção, trate isso melhor.
            # Se o cliente envia WAV, pode não precisar desta conversão explícita
            # se SpeechRecognition já espera um stream WAV.
            pass # Deixa para o SpeechRecognition tentar

        # Exportar para WAV em memória se não for já (SpeechRecognition lida melhor com WAV)
        wav_em_memoria = io.BytesIO()
        # Se som_audio foi carregado, exporte. Senão, o stream original é usado.
        if 'som_audio' in locals() and som_audio:
            som_audio.export(wav_em_memoria, format="wav")
            wav_em_memoria.seek(0)
            source_stream = wav_em_memoria
            print("Áudio convertido/exportado para WAV em memória.")
        else:
            # Se pydub falhou e estamos assumindo que o cliente enviou WAV:
            stream_audio.seek(0) # Garante que o stream está no início
            source_stream = stream_audio
            print("Usando stream original, assumindo que é WAV compatível.")


        with sr.AudioFile(source_stream) as source:
            dados_audio = recognizer.record(source)
            print("Dados de áudio gravados pelo SpeechRecognition.")

        texto_transcrito = recognizer.recognize_google(dados_audio, language="pt-BR")
        print(f"Transcrição: {texto_transcrito}")
        return {"nome_arquivo": nome_arquivo, "transcricao": texto_transcrito}

    except sr.UnknownValueError:
        print("Google Speech Recognition não conseguiu entender o áudio.")
        raise HTTPException(status_code=400, detail="O Google Speech Recognition não conseguiu entender o áudio.")
    except sr.RequestError as e:
        print(f"Erro no serviço Google Speech Recognition: {e}")
        raise HTTPException(status_code=503, detail=f"Não foi possível solicitar resultados do serviço Google Speech Recognition; {e}")
    except Exception as e:
        print(f"Erro inesperado ao processar o áudio: {e}")
        import traceback
        traceback.print_exc()
        raise HTTPException(status_code=500, detail=f"Erro ao processar o áudio: {str(e)}")

@app.get("/")
async def root():
    return {"message": "API de Transcrição. Use /docs para ver os endpoints."}

# if __name__ == "__main__":
#     uvicorn.run(app, host="0.0.0.0", port=8000)
# Rode com: uvicorn main:app --reload
```

**Execute a API Python:**
`uvicorn main:app --reload`

**2. Lado do Cliente (Exemplo Básico em HTML e JavaScript)**

Crie um arquivo `index.html` (ou sirva-o de alguma forma se estiver usando um framework frontend). Este código é ilustrativo e pode precisar de ajustes.

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Transcrição de Microfone</title>
    <style>
        body { font-family: sans-serif; margin: 20px; }
        button { padding: 10px 15px; margin: 5px; cursor: pointer; }
        #transcription { margin-top: 20px; padding: 10px; border: 1px solid #ccc; min-height: 50px; }
        .status { font-style: italic; color: #555; }
    </style>
</head>
<body>
    <h1>Transcrição de Áudio do Microfone</h1>

    <button id="startRecord">Iniciar Gravação</button>
    <button id="stopRecord" disabled>Parar Gravação e Transcrever</button>

    <p class="status" id="status">Pronto para gravar.</p>
    <h2>Transcrição:</h2>
    <div id="transcription"></div>

    <script>
        const startRecordButton = document.getElementById('startRecord');
        const stopRecordButton = document.getElementById('stopRecord');
        const transcriptionDiv = document.getElementById('transcription');
        const statusDiv = document.getElementById('status');

        let mediaRecorder;
        let audioChunks = [];
        // Tentar com 'audio/wav' primeiro, fallback para 'audio/webm' ou 'audio/ogg' que são mais comuns
        // O backend precisará lidar com o formato enviado.
        const mimeTypes = ['audio/wav', 'audio/webm;codecs=opus', 'audio/ogg;codecs=opus', 'audio/webm', 'audio/ogg'];
        let selectedMimeType = '';

        // Descobrir um MIME type suportado
        for (const mimeType of mimeTypes) {
            if (MediaRecorder.isTypeSupported(mimeType)) {
                selectedMimeType = mimeType;
                console.log("Usando MIME type:", selectedMimeType);
                break;
            }
        }
        if (!selectedMimeType) {
            statusDiv.textContent = "Nenhum formato de gravação suportado pelo navegador.";
            startRecordButton.disabled = true;
        }


        startRecordButton.onclick = async () => {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                mediaRecorder = new MediaRecorder(stream, { mimeType: selectedMimeType });

                mediaRecorder.ondataavailable = event => {
                    audioChunks.push(event.data);
                };

                mediaRecorder.onstart = () => {
                    statusDiv.textContent = 'Gravando...';
                    startRecordButton.disabled = true;
                    stopRecordButton.disabled = false;
                    transcriptionDiv.textContent = '';
                    audioChunks = []; // Limpa chunks anteriores
                };

                mediaRecorder.onstop = async () => {
                    statusDiv.textContent = 'Gravação parada. Enviando para transcrição...';
                    startRecordButton.disabled = false;
                    stopRecordButton.disabled = true;

                    // Determina a extensão do arquivo com base no MIME type
                    let fileExtension = ".bin"; // Genérico
                    if (selectedMimeType.includes("wav")) fileExtension = ".wav";
                    else if (selectedMimeType.includes("webm")) fileExtension = ".webm";
                    else if (selectedMimeType.includes("ogg")) fileExtension = ".ogg";

                    const audioBlob = new Blob(audioChunks, { type: selectedMimeType });
                    const audioFile = new File([audioBlob], `recording${fileExtension}`, { type: selectedMimeType });

                    const formData = new FormData();
                    formData.append('arquivo_audio', audioFile);

                    try {
                        const response = await fetch('http://127.0.0.1:8000/transcrever_microfone/', {
                            method: 'POST',
                            body: formData
                        });

                        if (response.ok) {
                            const result = await response.json();
                            transcriptionDiv.textContent = result.transcricao;
                            statusDiv.textContent = 'Transcrição recebida.';
                        } else {
                            const errorResult = await response.json();
                            transcriptionDiv.textContent = `Erro: ${response.status} - ${errorResult.detail || 'Falha ao transcrever'}`;
                            statusDiv.textContent = 'Erro na transcrição.';
                        }
                    } catch (error) {
                        console.error('Erro ao enviar áudio:', error);
                        transcriptionDiv.textContent = 'Erro de rede ou API indisponível.';
                        statusDiv.textContent = 'Erro de conexão.';
                    }
                };
                mediaRecorder.start(); // Inicia a gravação
            } catch (error) {
                console.error('Erro ao acessar microfone:', error);
                statusDiv.textContent = 'Erro ao acessar microfone. Verifique as permissões.';
                transcriptionDiv.textContent = `Erro: ${error.message}`;
            }
        };

        stopRecordButton.onclick = () => {
            if (mediaRecorder && mediaRecorder.state === 'recording') {
                mediaRecorder.stop();
            }
        };
    </script>
</body>
</html>
```

**Para testar `index.html`:**
*   Salve o código HTML acima como `index.html`.
*   Abra este arquivo diretamente no seu navegador (ex: `file:///caminho/para/seu/index.html`).
*   O navegador provavelmente pedirá permissão para usar o microfone. Conceda.
*   Clique em "Iniciar Gravação", fale algo, e depois clique em "Parar Gravação e Transcrever".
*   Verifique o console do navegador (F12) e o console do servidor Uvicorn para logs.

**Considerações Importantes:**

*   **Formato do Áudio:** `MediaRecorder` no navegador geralmente grava em formatos como WebM (com codec Opus ou VP8/VP9) ou Ogg. Alguns navegadores podem suportar WAV, mas não é universal. A API backend (`pydub`) precisa ser capaz de ler o formato que o cliente envia. O código `pydub` no backend tenta ser flexível, mas se o cliente sempre enviasse WAV, seria mais simples.
*   **`ffmpeg`:** `pydub` depende do `ffmpeg` para converter muitos formatos. Certifique-se de que ele está instalado no ambiente onde a API Python roda e acessível no PATH.
*   **HTTPS:** Para `getUserMedia` (acesso ao microfone), a maioria dos navegadores modernos exige que a página seja servida via HTTPS, exceto para `localhost` ou `127.0.0.1`. Se você for implantar isso, precisará de HTTPS.
*   **Erro `No format specified for MimeType...` no `pydub`:** Se `pydub` não conseguir inferir o formato (ex: o `MediaRecorder` envia um blob `audio/webm` mas `pydub` não o reconhece sem a extensão no nome do arquivo ou `format` explícito), você pode precisar:
    *   Garantir que o cliente envie um nome de arquivo com a extensão correta (ex: `recording.webm`). O `UploadFile` do FastAPI usa esse nome.
    *   Ou, na API, tentar especificar o formato para `AudioSegment.from_file(stream_audio, format="webm")` se você souber que é WebM.
*   **Real-time vs. Batch:** Este exemplo é "batch". Para streaming real-time, você usaria WebSockets. O cliente enviaria pequenos chunks de áudio continuamente, e o servidor os processaria. A biblioteca `SpeechRecognition` tem alguma capacidade de lidar com streams, mas a integração é mais complexa. Whisper geralmente espera um arquivo completo, então para streaming com Whisper, você precisaria de uma lógica de buffer e de decidir quando transcrever os segmentos acumulados.

Este exemplo ilustra o fluxo básico. Em uma aplicação real, você adicionaria mais tratamento de erros, feedback ao usuário e, possivelmente, exploraria WebSockets para uma experiência mais fluida se o "real-time" for crucial.
