FROM python:3.12-slim

WORKDIR /app
COPY ETL.py /app/ETL.py

RUN pip install --no-cache-dir requests

CMD ["python", "/app/ETL.py"]
