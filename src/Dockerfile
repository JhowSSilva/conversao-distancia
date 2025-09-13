FROM python:3.11-slim

WORKDIR /app

# Create a non-root user
RUN adduser --system --group appuser

# Copy requirements and install dependencies
COPY --chown=appuser:appuser requirements.txt ./requirements.txt
RUN pip install -r requirements.txt

# Copy the rest of the application code
COPY --chown=appuser:appuser . .

# Switch to the non-root user
USER appuser

EXPOSE 8000

CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "main:app"]