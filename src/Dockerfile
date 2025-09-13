FROM python

WORKDIR /app

COPY .requirements.txt ./requierments.txt

RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000" "main:app"]
