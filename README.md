# Semicolon

A file upload service that handles network interruptions gracefully. Upload large files with automatic resume capability when connections drop.

**Live Demo:** [semicolon-chi.vercel.app](https://semicolon-chi.vercel.app)  
**Demo Account:** `demo@semicolon.app` / `demo123456`

Built with Django REST Framework and Next.js during January-March 2026 as part of my full-stack development learning journey.

---

## What It Does

Handles file uploads up to 500MB with automatic resume on network failure. Files are split into 1MB chunks and uploaded sequentially - if your connection drops at 150MB, the upload continues from 150MB when reconnected.

**Core Features:**
- Chunked uploads with resume capability
- JWT authentication with HTTP-only cookies
- Real-time upload progress tracking  
- Rate limiting to prevent abuse
- File download management

---

## Technical Implementation

### Backend (Django REST Framework)
- **Chunked Upload Protocol**: Files split into 1MB chunks, uploaded via HTTP PATCH requests with Upload-Offset headers
- **Database Optimization**: Reduced writes by 90% (updates every 10MB instead of every chunk)
- **Authentication**: JWT tokens stored in HTTP-only secure cookies to prevent XSS attacks
- **Rate Limiting**: Redis-backed limits (10 uploads/hour, 50 downloads/hour per user)
- **Input Validation**: File type whitelist, size limits, path traversal prevention

### Frontend (Next.js 14)
- **Progress Tracking**: Real-time progress bar with percentage and MB uploaded
- **Retry Logic**: Automatic retry with exponential backoff (1s, 2s, 3s delays)
- **Protected Routes**: Automatic redirect to login for authenticated pages
- **TypeScript**: Type-safe API calls and component props

### Infrastructure
- **Backend**: Railway with PostgreSQL and Redis
- **Frontend**: Vercel with edge functions
- **Containerization**: Docker for consistent deployments
- **Database**: PostgreSQL with connection pooling

---

## How Resumable Uploads Work

The system uses offset-based chunk writing:

1. **Initialize**: Frontend requests upload session, backend creates empty file
2. **Upload Chunks**: Each 1MB chunk sent with its offset position (0MB, 1MB, 2MB...)
3. **Write at Offset**: Backend uses `file.seek(offset)` to write chunk at correct position
4. **Network Failure**: If connection drops at 150MB, client resumes from chunk 151
5. **Completion**: When all chunks uploaded, status marked as "COMPLETED"

This approach allows chunks to be uploaded in any order and enables true resume capability.

---

## Local Development

### Prerequisites
- Python 3.12+
- Node.js 18+
- PostgreSQL (or use SQLite for dev)
- Redis (optional, for rate limiting)

### Backend Setup

```bash
cd semicolon-backend

# Create virtual environment
python -m venv venv
venv\Scripts\activate  # Windows
# source venv/bin/activate  # macOS/Linux

# Install dependencies
pip install -r requirements.txt

# Copy environment file
copy .env.example .env.local
# Edit .env.local with your settings

# Run migrations
python manage.py migrate

# Create demo user
python manage.py create_demo_user

# Start server
python manage.py runserver
```

Backend runs at `http://localhost:8000`

### Frontend Setup

```bash
cd semicolon-frontend

# Install dependencies
npm install

# Copy environment file
copy .env.example .env.local
# Edit .env.local: NEXT_PUBLIC_BACKEND_URL=http://localhost:8000

# Start development server
npm run dev
```

Frontend runs at `http://localhost:3000`

---

## Docker Setup

```bash
# Copy environment files
copy semicolon-backend\.env.example semicolon-backend\.env.local
copy semicolon-frontend\.env.example semicolon-frontend\.env.local

# Start all services
docker-compose up -d

# Create demo user
docker-compose exec backend python manage.py create_demo_user
```

Access at `http://localhost:3000`

---

## Key Decisions & Tradeoffs

### Why HTTP-Only Cookies Over localStorage?

**Security.** If an attacker injects malicious JavaScript (XSS), they can read `localStorage` but not HTTP-only cookies. The browser sends cookies automatically with requests, but JavaScript cannot access them.

```javascript
// XSS attack scenario
localStorage.getItem('token')  // ✅ Works - token stolen
document.cookie                 // ❌ Empty - httponly cookie invisible
```

### Why 1MB Chunks?

Initially tried 5MB chunks but encountered HTTP/2 protocol errors on Railway's proxy. 1MB chunks provide:
- Smooth progress updates (100 updates for 100MB file)
- Better compatibility with cloud infrastructure
- Faster retry on individual chunk failure

### Why Batch Database Updates?

Updating the database on every chunk (228 writes for 228MB) was slow. Now updates every 10MB or on completion (23 writes) - 90% reduction in database I/O.

### Why Local File Storage?

For this portfolio project, local disk storage is sufficient. For production scale:
- Use AWS S3 or equivalent cloud storage
- Generate presigned URLs for direct client-to-S3 uploads
- Removes Django from the upload path entirely

---

## Project Structure

```
semicolon/
├── semicolon-backend/
│   ├── core/                 # Django settings & config
│   ├── users/                # Authentication
│   ├── upload/               # Upload/download logic
│   ├── requirements.txt
│   └── Dockerfile
│
├── semicolon-frontend/
│   ├── app/                  # Next.js pages
│   ├── components/           # React components
│   ├── lib/                  # API client & utilities
│   ├── package.json
│   └── Dockerfile
│
├── docker-compose.yml
├── PROJECT_HIGHLIGHTS.md     # Resume content & metrics
└── STUDY_GUIDE.md           # Deep dive explanations
```

---

## What I Learned

**Full-Stack Development:**
- Building RESTful APIs with Django REST Framework
- Client-side state management with React Context
- JWT authentication flow with refresh tokens
- Handling file I/O at scale

**Security:**
- XSS prevention with HTTP-only cookies
- CSRF protection for state-changing requests
- Input validation and sanitization
- Rate limiting strategies

**Performance:**
- Database query optimization (select_related, only())
- Reducing I/O operations (batch updates)
- Chunking strategies for large file transfers
- Retry logic with exponential backoff

**DevOps:**
- Docker containerization
- Environment-based configuration
- Database migrations in production
- Cloud deployment on Railway and Vercel

**Problem Solving:**
- Debugging HTTP/2 protocol errors (reduced chunk size)
- Handling Railway DNS issues (worked with support)
- Optimizing database writes (90% reduction)
- Implementing proper error handling

---

## Known Limitations

**Current Implementation:**
- Files stored on local disk (ephemeral on Railway volumes)
- Single file upload at a time
- No file preview or thumbnails
- No sharing capabilities
- No virus scanning

**Would Add for Production:**
- AWS S3 integration with presigned URLs
- Multiple concurrent uploads with queue
- Image thumbnails using Pillow
- Shareable links with expiry
- ClamAV for virus scanning
- Email notifications on completion
- Upload analytics dashboard
- Comprehensive test suite (pytest + Jest)

---

## Deployment

### Backend (Railway)
1. Create new project from GitHub repo
2. Add PostgreSQL and Redis services
3. Set environment variables (see `.env.example`)
4. Railway auto-deploys on push to main

### Frontend (Vercel)
1. Import GitHub repo
2. Set `NEXT_PUBLIC_BACKEND_URL` environment variable
3. Vercel auto-deploys on push to main

---

## Resources

- **[PROJECT_HIGHLIGHTS.md](./PROJECT_HIGHLIGHTS.md)** - Resume bullets, metrics, and interview prep
- **[STUDY_GUIDE.md](./STUDY_GUIDE.md)** - Deep explanations of how everything works
- **[TUS Protocol](https://tus.io/)** - Inspiration for this project

---

## License

MIT - Use this for learning or as a starting point for your own projects.

---

## Contact

Built by Vikas Singh as a portfolio project.

- GitHub: [@vikas90244](https://github.com/vikas90244)
- Project Link: [github.com/vikas90244/semicolon](https://github.com/vikas90244/semicolon)

---

*If this project helped you understand chunked uploads, file I/O, or full-stack development, consider giving it a star!*
