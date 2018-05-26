FROM python:3.6.0-alpine

ENV POST_DATABASE_HOST post_db
ENV POST_DATABASE posts

WORKDIR /app

RUN apk add --no-cache gcc musl-dev
COPY requirements.txt .
RUN pip install -r /app/requirements.txt

ENTRYPOINT ["python3", "post_app.py"]

COPY . .

