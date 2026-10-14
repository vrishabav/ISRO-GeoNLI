# Website Backend

FastAPI backend for the ISRO-GeoNLI web interface. It handles login, stores chat sessions and messages in MongoDB, stores images in AWS S3, and forwards each query to the ISRO-GeoNLI model API.

## Architecture

```
website-backend/
├── main.py              # FastAPI app, CORS, router registration
├── db.py                # MongoDB (Motor) connection
├── routers/
│   ├── login.py         # POST /api/login
│   ├── chat.py          # POST /api/chat
│   ├── chatHistory.py   # GET /api/chat-history
│   └── getHistory.py    # GET /api/get-history
```

## Prerequisites

- Python 3.10+
- MongoDB instance (local or cloud)
- AWS S3 bucket (for image storage)
- The ISRO-GeoNLI model API (`app_prod.py`) running and reachable

## Installation

```bash
cd website-backend
pip install fastapi uvicorn motor pymongo "python-jose[cryptography]" "pydantic[email]" python-multipart boto3 python-dotenv httpx requests pillow
```

Create a `.env` file in `website-backend/`:

```env
# MongoDB
DATABASE_URL=mongodb://localhost:27017

# AWS S3
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_REGION=us-east-1
AWS_BUCKET_NAME=your_bucket_name

# JWT
JWT_SECRET=your_jwt_secret_key
JWT_ALGORITHM=HS256

# ISRO-GeoNLI model API (base URL, without /query)
ISRO_API_URL=http://localhost:8080
```

## Running the Server

```bash
uvicorn main:app --host 0.0.0.0 --port 8001
```

## API Endpoints

All routes are mounted under `/api`.

**POST /api/login**
- Request: `{"email": "user@example.com", "password": "password"}`
- Response: `{"access_token": "<jwt>", "token_type": "bearer"}`
- There is no signup endpoint. Users are created directly in the `users` collection.

**POST /api/chat** (requires `Authorization: Bearer <jwt>`)
- Form fields: `session_id`, `query`, `spatial_resolution_m`, and either `image` (file) or `image_url`
- The first message of a session must include `image`. It is uploaded to S3 and stored on the session.
- The query is sent to `{ISRO_API_URL}/query`. An annotated image in the reply is uploaded to S3.
- Response: `{"text": "...", "image": "<s3 url or empty string>"}`

**GET /api/chat-history** (requires `Authorization: Bearer <jwt>`)
- Response: list of `{"sessionId": "...", "title": "..."}` for the logged-in user

**GET /api/get-history?sessionId=...**
- Response: `{"messages": [{"role", "type", "content"}], "image": {"image_url": "..."}}`
- This route does not check the JWT.

## Database Schema

**users**
```json
{
  "email": "user@example.com",
  "password": "plaintext password"
}
```

**sessions**
```json
{
  "sessionId": "uuid",
  "email": "user@example.com",
  "title": "first query of the session",
  "imageURL": "https://<bucket>.s3.<region>.amazonaws.com/<file>",
  "createdAt": 1700000000
}
```

**messages**
```json
{
  "sessionId": "uuid",
  "role": "user | assistant",
  "type": "text | image",
  "content": "message text or S3 image URL",
  "timestamp": 1700000000
}
```

## CORS

`main.py` allows requests only from:
```python
origins = [
    "https://isrogeonli.in:5173",
]
```
Add your frontend URL (for example `http://localhost:5173`) for local development.

## Known Limitations

1. Passwords are stored and compared in plaintext. They are not hashed.
2. JWTs have no expiry claim.
3. `/api/get-history` is not authenticated.
4. Images are uploaded to S3 with a `public-read` ACL.

## License

Part of the ISRO-GeoNLI project.
