FROM python:3.11-slim

# Crie o usuário e grupo para rodar a aplicação
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copie requirements para dentro da imagem
COPY --chown=appuser:appuser requirements.txt ./requirements.txt

RUN pip install -r requirements.txt

COPY --chown=appuser:appuser . .

USER appuser

CMD ["python", "main.py"]
