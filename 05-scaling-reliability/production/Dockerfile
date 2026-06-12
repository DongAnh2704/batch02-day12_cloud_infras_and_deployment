FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY 02-docker/production/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
RUN pip install --no-cache-dir redis

# Copy app code
COPY 05-scaling-reliability/production/app.py .

# Copy utils
RUN mkdir -p utils
COPY utils/mock_llm.py utils/

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
