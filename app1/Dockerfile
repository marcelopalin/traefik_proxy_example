FROM tiangolo/uvicorn-gunicorn-fastapi:python3.9

COPY . /app
CMD ["uvicorn", "app1.main:app", "--host", "0.0.0.0", "--port", "8000"]
