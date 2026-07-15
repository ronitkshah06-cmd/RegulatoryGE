# Regulatory Intelligence Assistant — Deployment & Operations Guide

## Overview

The Regulatory Intelligence Assistant is a production-ready Streamlit application that analyzes regulatory/SOP excerpts against product change scenarios. It provides structured, source-grounded pre-review analysis with enterprise-grade features including authentication, audit logging, rate limiting, and compliance support for HIPAA and GDPR.

**Status**: Production-Ready (v2.0+)

## Key Features

### Core Analysis
- **Structured Analysis**: Returns JSON-formatted risk assessment with evidence basis
- **Source Grounding**: Enforces strict "answer only from pasted text" contract
- **Risk Classification**: Low, Medium, High, or Unclear ratings
- **Multi-format Export**: Markdown, JSON, CSV, PDF

### Security & Compliance
- **User Authentication**: Username/password with session management
- **Audit Logging**: Complete audit trail for HIPAA/GDPR compliance
- **Input Validation & Sanitization**: Prevents injection attacks
- **Rate Limiting**: Per-user API call throttling (default 60/hour)
- **Encryption Ready**: Support for encrypted secrets and data at rest

### Data Management
- **Analysis Persistence**: SQLite/PostgreSQL database integration
- **History Tracking**: Full analysis history per user
- **Automatic Cleanup**: Log rotation and retention policies

### Administration
- **Admin Dashboard**: System health and audit log viewing
- **Feature Toggles**: Environment-based feature configuration
- **Flexible Deployment**: Streamlit Cloud, Docker, AWS, Azure, VPS

## Quick Start

### Local Development

#### 1. Clone and Setup
```bash
git clone https://github.com/your-org/regulatory-intelligence-assistant.git
cd regulatory-intelligence-assistant

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

#### 2. Configure Environment
```bash
cp .env.example .env

# Edit .env:
# - Add your OPENROUTER_API_KEY from https://openrouter.ai/keys
# - Enable auth: ENABLE_AUTH=true
# - Enable audit logging: ENABLE_AUDIT_LOG=true
```

#### 3. Run Locally
```bash
streamlit run app.py
```

The app opens at http://localhost:8501

### Configuration

All settings are environment-variable driven via `.env` (local) or `.streamlit/secrets.toml` (deployed).

#### Security Settings
```env
SECRET_KEY=your-secret-key              # Must change in production
ENABLE_AUTH=true                        # Require login
AUTH_SECRET=your-auth-secret            # Session encryption key
ENABLE_ENCRYPTION=true                  # Enable data encryption at rest
```

#### Feature Flags
```env
ENABLE_PERSISTENCE=true                 # Save to database
ENABLE_AUDIT_LOG=true                   # Log all actions
ENABLE_RATE_LIMITING=true               # Throttle API calls
RATE_LIMIT_CALLS_PER_HOUR=60            # Max calls per hour
```

#### Compliance
```env
HIPAA_MODE=false                        # HIPAA compliance mode
GDPR_MODE=false                         # GDPR data handling
LOG_RETENTION_DAYS=90                   # Audit log retention
```

#### Database
```env
# SQLite (default, local development only)
DATABASE_URL=sqlite:///regulatory_assistant.db

# PostgreSQL (production recommended)
DATABASE_URL=postgresql://user:pass@localhost:5432/regulatory_db
```

## Deployment

### Option 1: Streamlit Community Cloud (Recommended for Getting Started)

1. **Push to GitHub**
   ```bash
   git add .
   git commit -m "Initial commit"
   git push origin main
   ```

2. **Deploy via Streamlit Cloud**
   - Go to https://share.streamlit.io
   - Click "New app"
   - Select your repo, branch `main`, and file `app.py`

3. **Configure Secrets**
   - Under **Advanced settings → Secrets**, add:
   ```toml
   OPENROUTER_API_KEY = "sk-or-v1-your-real-key"
   ENABLE_AUTH = "true"
   AUTH_SECRET = "your-auth-secret"
   ENABLE_PERSISTENCE = "false"  # SQLite not recommended for shared hosting
   ```

4. **Deploy** — Get a public `*.streamlit.app` URL

### Option 2: AWS (Production Recommended)

#### Deploy on AWS App Runner (Easiest)

1. **Create Dockerfile**
   ```dockerfile
   FROM python:3.11-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install -r requirements.txt
   COPY . .
   EXPOSE 8501
   CMD ["streamlit", "run", "app.py", "--server.port", "8501", "--server.address", "0.0.0.0"]
   ```

2. **Push to ECR or DockerHub**
   ```bash
   docker build -t regulatory-assistant:latest .
   docker push your-registry/regulatory-assistant:latest
   ```

3. **Create AWS App Runner Service**
   - Source: Container image registry
   - Image URI: `your-registry/regulatory-assistant:latest`
   - Environment variables:
     - `OPENROUTER_API_KEY`: (from AWS Secrets Manager)
     - `ENABLE_AUTH`: `true`
     - `DATABASE_URL`: (RDS PostgreSQL endpoint)
   - Auto-deploy on push: Enabled

4. **Setup RDS PostgreSQL**
   - Create RDS instance with PostgreSQL 15+
   - Create database: `regulatory_db`
   - The app will auto-create tables on first run

#### Deploy on EC2 (Manual Control)

1. **Launch EC2 instance**
   - AMI: Ubuntu 22.04
   - Instance: t3.medium or larger (Python + DB)
   - Security Group: Allow ports 80, 443, 22

2. **Setup on EC2**
   ```bash
   sudo apt update && sudo apt install -y python3-pip python3-venv postgresql

   # Clone repo
   git clone https://github.com/your-org/regulatory-intelligence-assistant.git
   cd regulatory-intelligence-assistant

   # Setup Python
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt

   # Setup PostgreSQL
   sudo -u postgres createdb regulatory_db
   sudo -u postgres createuser app_user
   # Set password and grant permissions
   ```

3. **Setup Systemd Service**
   ```ini
   # /etc/systemd/system/regulatory-assistant.service
   [Unit]
   Description=Regulatory Intelligence Assistant
   After=network.target postgresql.service

   [Service]
   User=ubuntu
   WorkingDirectory=/home/ubuntu/regulatory-intelligence-assistant
   Environment="OPENROUTER_API_KEY=sk-or-v1-..."
   Environment="ENABLE_AUTH=true"
   Environment="DATABASE_URL=postgresql://app_user:pass@localhost/regulatory_db"
   ExecStart=/home/ubuntu/regulatory-intelligence-assistant/.venv/bin/streamlit run app.py \
       --server.port 8501 --server.address 0.0.0.0
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```

4. **Setup Nginx Reverse Proxy**
   ```nginx
   # /etc/nginx/sites-available/regulatory-assistant
   upstream regulatory_app {
       server 127.0.0.1:8501;
   }

   server {
       listen 80;
       listen 443 ssl http2;
       server_name your-domain.com;

       ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

       location / {
           proxy_pass http://regulatory_app;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "upgrade";
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

5. **Start Service**
   ```bash
   sudo systemctl enable regulatory-assistant
   sudo systemctl start regulatory-assistant
   sudo nginx -t && sudo systemctl restart nginx
   ```

### Option 3: Azure App Service

1. **Create App Service Plan**
   ```bash
   az appservice plan create --name regulatory-plan \
     --resource-group your-rg --sku B2 --is-linux
   ```

2. **Deploy Container**
   ```bash
   az webapp create --resource-group your-rg \
     --plan regulatory-plan --name regulatory-app \
     --deployment-container-image-name your-registry/regulatory-assistant:latest
   ```

3. **Configure Application Settings**
   ```bash
   az webapp config appsettings set --resource-group your-rg \
     --name regulatory-app \
     --settings OPENROUTER_API_KEY=sk-or-v1-... ENABLE_AUTH=true
   ```

### Option 4: Docker Compose (Development/Testing)

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: regulatory_db
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: changeme
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  app:
    build: .
    ports:
      - "8501:8501"
    environment:
      DATABASE_URL: postgresql://app_user:changeme@postgres:5432/regulatory_db
      OPENROUTER_API_KEY: sk-or-v1-...
      ENABLE_AUTH: "true"
      ENABLE_PERSISTENCE: "true"
    depends_on:
      - postgres

volumes:
  postgres_data:
```

Run: `docker-compose up`

## Security & Compliance

### Authentication

- **Password Requirements**: 12+ chars, uppercase, lowercase, numbers, symbols
- **Session Timeout**: 24 hours
- **Password Hashing**: PBKDF2-SHA256 (production: use bcrypt/argon2)

To enable authentication:
```env
ENABLE_AUTH=true
AUTH_SECRET=your-secure-secret-key-minimum-32-chars
```

### Input Validation & Sanitization

- **Regulatory Text**: 50-50,000 characters
- **Scenario Question**: 10-5,000 characters
- **Null byte removal**: Automatic
- **XSS prevention**: All user input escaped
- **SQL injection prevention**: SQLAlchemy parameterized queries

### Audit Logging

All sensitive operations logged:
- User login/logout
- Analysis submissions
- API errors
- Rate limit violations
- Invalid inputs

Logs stored in `.logs/audit.log` (JSON format) and database.

Enable:
```env
ENABLE_AUDIT_LOG=true
LOG_RETENTION_DAYS=90
```

Example audit log entry:
```json
{
  "timestamp": "2026-07-15T14:30:00.123456",
  "event_type": "analysis_completed",
  "user_id": "123",
  "success": true,
  "details": {
    "risk_level": "Medium",
    "document_type": "FDA Guidance"
  }
}
```

### HIPAA Compliance

To enable HIPAA mode:
```env
HIPAA_MODE=true
```

This enables:
- Audit logging of all data access
- Encryption at rest for database
- Automatic log retention (90 days)
- User authentication requirement
- Rate limiting

**Note**: Full HIPAA compliance requires:
1. Business Associate Agreement (BAA) with OpenRouter
2. Data Encryption at rest and in transit (TLS 1.2+)
3. Access controls and authentication
4. Audit logging and monitoring
5. Regular security assessments

### GDPR Compliance

To enable GDPR mode:
```env
GDPR_MODE=true
```

This enables:
- Data export (user's data in standard format)
- Right to deletion (user account/data erasure)
- Consent tracking
- Data minimization

**Note**: Full GDPR compliance also requires:
1. Privacy policy and data processing agreement
2. Data Processing Agreement (DPA) with OpenRouter
3. User consent for data processing
4. Data retention and purge policies
5. Privacy impact assessments

### Rate Limiting

Prevents API abuse:
```env
ENABLE_RATE_LIMITING=true
RATE_LIMIT_CALLS_PER_HOUR=60
```

- Per-user or per-session throttling
- Exceeding limit returns 429 status
- Audit logged for security analysis

## Monitoring & Operations

### Log Files

- **Application Log**: `.logs/app.log` (rotated at 10MB, keeps 5 backups)
- **Audit Log**: `.logs/audit.log` (rotated at 10MB, keeps 10 backups)

### Database Maintenance

#### SQLite (Development)
```bash
# Backup
cp regulatory_assistant.db regulatory_assistant.db.backup

# Vacuum (optimize)
sqlite3 regulatory_assistant.db "VACUUM;"
```

#### PostgreSQL (Production)
```bash
# Backup
pg_dump regulatory_db > backup_$(date +%Y%m%d).sql

# Restore
psql regulatory_db < backup_YYYYMMDD.sql

# Vacuum
psql regulatory_db -c "VACUUM ANALYZE;"
```

### Monitoring Checklist

- [ ] API key rotation every 90 days
- [ ] Database backups daily (automated)
- [ ] Log monitoring for errors
- [ ] Rate limit metrics dashboarding
- [ ] Audit log review weekly
- [ ] Security updates monthly
- [ ] Load testing before peak periods

## Troubleshooting

### Common Issues

#### 1. "OPENROUTER_API_KEY is not configured"
**Solution**: Set environment variable or add to `.streamlit/secrets.toml`
```toml
OPENROUTER_API_KEY = "sk-or-v1-..."
```

#### 2. "Database connection failed"
**Solution**: Check DATABASE_URL and PostgreSQL credentials
```env
DATABASE_URL=postgresql://user:password@host:5432/dbname
```

#### 3. "Rate limit exceeded"
**Solution**: Wait for reset or increase `RATE_LIMIT_CALLS_PER_HOUR`
```env
RATE_LIMIT_CALLS_PER_HOUR=120
```

#### 4. "Authentication required" on Streamlit Cloud
**Solution**: Configure secrets in Streamlit Cloud dashboard
Settings → Advanced settings → Secrets

#### 5. Tables not created in database
**Solution**: The app auto-creates tables on first run. If issues:
```python
# Run manually in Python shell:
from src.database import get_db_manager
db = get_db_manager()
db.init_db()
```

## Testing

### Run Tests
```bash
pytest tests/ -v

# With coverage
pytest tests/ --cov=src --cov-report=html
```

### Key Test Files
- `tests/test_prompt_contract.py`: Verify analysis contract enforcement
- `tests/test_sample_data.py`: Validate sample scenarios
- Add `tests/test_auth.py` for authentication
- Add `tests/test_validation.py` for input validation
- Add `tests/test_export.py` for export formats

## API Reference (Future Enhancement)

A REST API version is planned. When implemented, it will support:

```bash
# Get analysis
POST /api/v1/analyze
{
  "document_type": "FDA Guidance",
  "regulatory_text": "...",
  "scenario_question": "..."
}

# Get user's analyses
GET /api/v1/analyses?limit=10

# Download analysis
GET /api/v1/analyses/{id}/export?format=json
```

## Support & Contributing

- **Issues**: Report bugs via GitHub Issues
- **Docs**: https://github.com/your-org/regulatory-intelligence-assistant/wiki
- **Contributing**: See CONTRIBUTING.md
- **License**: (Your license here)

## Version History

- **v2.0** (Current): Enterprise features (auth, persistence, audit)
- **v1.0**: Initial MVP with core analysis

---

**Last Updated**: July 15, 2026  
**Maintained By**: Your Organization  
**Status**: ✅ Production Ready
