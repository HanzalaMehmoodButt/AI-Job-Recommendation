# AI Job Recommendation System - Node.js Architecture & Database Schema

## 1. System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│  React/Vue Frontend  │  Mobile App  │  Third-party Integrations │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     API GATEWAY / LOAD BALANCER                 │
│                      (NGINX / AWS ALB)                          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     NODE.JS APPLICATION LAYER                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │   Auth      │  │    User     │  │    Job      │            │
│  │  Service    │  │  Service    │  │  Service    │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │Recommendation│  │    CV       │  │  Scraping   │            │
│  │   Engine    │  │  Generator  │  │  Service    │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      MESSAGE QUEUE LAYER                        │
│                    (Redis / Bull Queue)                         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      BACKGROUND WORKERS                         │
├─────────────────────────────────────────────────────────────────┤
│  ML Training  │  Job Scraping  │  Email Notifications  │  PDF Gen │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                               │
├─────────────────────────────────────────────────────────────────┤
│ PostgreSQL  │  Redis Cache  │  File Storage  │  ML Model Store  │
│  (Primary)  │   (Session)   │   (S3/Local)   │   (TensorFlow)   │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Node.js Application Structure

```
src/
├── config/
│   ├── database.js
│   ├── redis.js
│   ├── auth.js
│   └── environment.js
├── controllers/
│   ├── authController.js
│   ├── userController.js
│   ├── jobController.js
│   ├── recommendationController.js
│   └── cvController.js
├── services/
│   ├── authService.js
│   ├── userService.js
│   ├── jobService.js
│   ├── recommendationService.js
│   ├── cvService.js
│   ├── scrapingService.js
│   └── mlService.js
├── models/
│   ├── User.js
│   ├── Job.js
│   ├── Application.js
│   ├── Recommendation.js
│   └── CVTemplate.js
├── middleware/
│   ├── auth.js
│   ├── validation.js
│   ├── rateLimiting.js
│   └── errorHandler.js
├── routes/
│   ├── auth.js
│   ├── users.js
│   ├── jobs.js
│   ├── recommendations.js
│   └── cv.js
├── utils/
│   ├── logger.js
│   ├── helpers.js
│   ├── validators.js
│   └── mlUtils.js
├── workers/
│   ├── jobScraper.js
│   ├── mlTrainer.js
│   ├── emailWorker.js
│   └── pdfGenerator.js
└── app.js
```

## 3. Database Schema (PostgreSQL)

### Users Table
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    location JSONB, -- {city, state, country, coordinates}
    profile_picture_url VARCHAR(500),
    linkedin_url VARCHAR(500),
    github_url VARCHAR(500),
    portfolio_url VARCHAR(500),
    bio TEXT,
    career_level VARCHAR(50), -- junior, mid, senior, executive
    preferred_salary_min INTEGER,
    preferred_salary_max INTEGER,
    currency VARCHAR(3) DEFAULT 'USD',
    remote_preference VARCHAR(20), -- remote, hybrid, onsite, flexible
    willing_to_relocate BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    email_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Skills Table
```sql
CREATE TABLE skills (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    category VARCHAR(50), -- technical, soft, language, certification
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### User Skills Table (Many-to-Many)
```sql
CREATE TABLE user_skills (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    skill_id INTEGER REFERENCES skills(id) ON DELETE CASCADE,
    proficiency_level INTEGER CHECK (proficiency_level >= 1 AND proficiency_level <= 5),
    years_of_experience DECIMAL(3,1),
    is_primary BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, skill_id)
);
```

### Experience Table
```sql
CREATE TABLE experiences (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    company_name VARCHAR(200) NOT NULL,
    job_title VARCHAR(200) NOT NULL,
    employment_type VARCHAR(50), -- full-time, part-time, contract, freelance
    location JSONB,
    is_remote BOOLEAN DEFAULT FALSE,
    start_date DATE NOT NULL,
    end_date DATE,
    is_current BOOLEAN DEFAULT FALSE,
    description TEXT,
    achievements TEXT[],
    technologies_used INTEGER[], -- Array of skill IDs
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Education Table
```sql
CREATE TABLE education (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    institution_name VARCHAR(200) NOT NULL,
    degree VARCHAR(100),
    field_of_study VARCHAR(100),
    grade VARCHAR(20),
    start_date DATE,
    end_date DATE,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Companies Table
```sql
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) UNIQUE NOT NULL,
    description TEXT,
    industry VARCHAR(100),
    size VARCHAR(50), -- startup, small, medium, large, enterprise
    location JSONB,
    website VARCHAR(500),
    logo_url VARCHAR(500),
    culture_tags TEXT[], -- Array of culture descriptors
    glassdoor_rating DECIMAL(2,1),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Jobs Table
```sql
CREATE TABLE jobs (
    id SERIAL PRIMARY KEY,
    external_id VARCHAR(100), -- ID from job board
    title VARCHAR(200) NOT NULL,
    company_id INTEGER REFERENCES companies(id),
    company_name VARCHAR(200), -- Fallback if company not in our DB
    description TEXT NOT NULL,
    requirements TEXT,
    benefits TEXT,
    employment_type VARCHAR(50),
    experience_level VARCHAR(50),
    location JSONB,
    is_remote BOOLEAN DEFAULT FALSE,
    salary_min INTEGER,
    salary_max INTEGER,
    currency VARCHAR(3) DEFAULT 'USD',
    posted_date TIMESTAMP,
    expires_date TIMESTAMP,
    application_url VARCHAR(500),
    source VARCHAR(100), -- indeed, linkedin, company_website
    is_active BOOLEAN DEFAULT TRUE,
    view_count INTEGER DEFAULT 0,
    application_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Job Skills Table (Many-to-Many)
```sql
CREATE TABLE job_skills (
    id SERIAL PRIMARY KEY,
    job_id INTEGER REFERENCES jobs(id) ON DELETE CASCADE,
    skill_id INTEGER REFERENCES skills(id) ON DELETE CASCADE,
    is_required BOOLEAN DEFAULT TRUE,
    importance_weight DECIMAL(3,2) DEFAULT 1.0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(job_id, skill_id)
);
```

### Applications Table
```sql
CREATE TABLE applications (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    job_id INTEGER REFERENCES jobs(id) ON DELETE CASCADE,
    status VARCHAR(50) DEFAULT 'applied', -- applied, viewed, interview, rejected, offer, accepted
    applied_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    cv_version_id INTEGER, -- Reference to specific CV used
    cover_letter TEXT,
    notes TEXT,
    interview_dates JSONB[], -- Array of interview schedules
    feedback TEXT,
    salary_offered INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, job_id)
);
```

### Recommendations Table
```sql
CREATE TABLE recommendations (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    job_id INTEGER REFERENCES jobs(id) ON DELETE CASCADE,
    match_score DECIMAL(5,4), -- 0.0000 to 1.0000
    reasons JSONB, -- {"skills_match": 0.85, "location_match": 0.9, "experience_match": 0.75}
    algorithm_version VARCHAR(20),
    is_viewed BOOLEAN DEFAULT FALSE,
    is_saved BOOLEAN DEFAULT FALSE,
    is_dismissed BOOLEAN DEFAULT FALSE,
    user_feedback INTEGER, -- 1-5 rating
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, job_id)
);
```

### CV Templates Table
```sql
CREATE TABLE cv_templates (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    template_data JSONB, -- Template structure and styling
    preview_image_url VARCHAR(500),
    category VARCHAR(50), -- modern, classic, creative, ats-friendly
    is_premium BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### User CVs Table
```sql
CREATE TABLE user_cvs (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    template_id INTEGER REFERENCES cv_templates(id),
    name VARCHAR(200) NOT NULL,
    cv_data JSONB, -- Complete CV content and customizations
    is_default BOOLEAN DEFAULT FALSE,
    download_count INTEGER DEFAULT 0,
    last_downloaded TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### User Preferences Table
```sql
CREATE TABLE user_preferences (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE UNIQUE,
    job_alert_frequency VARCHAR(20) DEFAULT 'daily', -- none, daily, weekly
    preferred_industries TEXT[],
    excluded_companies INTEGER[], -- Array of company IDs to exclude
    notification_settings JSONB, -- Email, push, SMS preferences
    privacy_settings JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 4. Key Node.js Services Architecture

### Authentication Service
```javascript
class AuthService {
    async register(userData) { /* JWT token generation */ }
    async login(email, password) { /* Password verification */ }
    async refreshToken(token) { /* Token refresh */ }
    async resetPassword(email) { /* Password reset flow */ }
    async verifyEmail(token) { /* Email verification */ }
}
```

### Recommendation Engine Service
```javascript
class RecommendationService {
    async generateRecommendations(userId) {
        // 1. Get user profile and preferences
        // 2. Calculate skill matching scores
        // 3. Apply location and salary filters
        // 4. Use collaborative filtering
        // 5. Rank and return top matches
    }
    
    async updateUserFeedback(userId, jobId, feedback) {
        // Update ML model with user feedback
    }
}
```

### ML Service Integration
```javascript
class MLService {
    async trainRecommendationModel() {
        // Background training of recommendation models
    }
    
    async calculateJobMatch(userProfile, jobData) {
        // Real-time job matching calculation
    }
    
    async extractSkillsFromJobDescription(description) {
        // NLP-based skill extraction
    }
}
```

## 5. API Endpoints Structure

### Authentication Routes
- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/logout`
- `POST /api/auth/forgot-password`
- `POST /api/auth/reset-password`

### User Management Routes
- `GET /api/users/profile`
- `PUT /api/users/profile`
- `POST /api/users/skills`
- `PUT /api/users/skills/:id`
- `DELETE /api/users/skills/:id`
- `POST /api/users/experience`
- `PUT /api/users/experience/:id`

### Job Routes
- `GET /api/jobs` (with filtering and pagination)
- `GET /api/jobs/:id`
- `POST /api/jobs/:id/apply`
- `GET /api/jobs/:id/similar`

### Recommendation Routes
- `GET /api/recommendations`
- `POST /api/recommendations/:id/feedback`
- `PUT /api/recommendations/:id/save`
- `PUT /api/recommendations/:id/dismiss`

### CV Routes
- `GET /api/cv/templates`
- `POST /api/cv/generate`
- `GET /api/cv/:id`
- `PUT /api/cv/:id`
- `POST /api/cv/:id/download`

## 6. Technology Stack Details

### Core Dependencies
```json
{
  "express": "^4.18.2",
  "postgres": "^3.4.3",
  "redis": "^4.6.10",
  "bcryptjs": "^2.4.3",
  "jsonwebtoken": "^9.0.2",
  "joi": "^17.11.0",
  "bull": "^4.11.4",
  "puppeteer": "^21.5.2",
  "cheerio": "^1.0.0-rc.12",
  "nodemailer": "^6.9.7",
  "winston": "^3.11.0",
  "helmet": "^7.1.0",
  "cors": "^2.8.5",
  "express-rate-limit": "^7.1.5"
}
```

### ML Integration
```json
{
  "@tensorflow/tfjs-node": "^4.15.0",
  "natural": "^6.5.0",
  "ml-matrix": "^6.10.7",
  "compromise": "^14.10.0"
}
```

## 7. Performance Optimization

### Database Indexing Strategy
```sql
-- User lookups
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_location ON users USING GIN(location);

-- Job searches
CREATE INDEX idx_jobs_title ON jobs(title);
CREATE INDEX idx_jobs_location ON jobs USING GIN(location);
CREATE INDEX idx_jobs_posted_date ON jobs(posted_date DESC);
CREATE INDEX idx_jobs_active ON jobs(is_active) WHERE is_active = true;

-- Recommendations
CREATE INDEX idx_recommendations_user_score ON recommendations(user_id, match_score DESC);
CREATE INDEX idx_recommendations_created ON recommendations(created_at DESC);

-- Skills matching
CREATE INDEX idx_user_skills_user ON user_skills(user_id);
CREATE INDEX idx_job_skills_job ON job_skills(job_id);
```

### Caching Strategy
- **Redis Cache**: User sessions, frequent job searches, recommendation cache
- **Application-level**: Skills dictionary, company data, CV templates
- **CDN**: Static assets, generated PDFs, profile images

## 8. Security Implementation

### Data Protection
- Password hashing with bcrypt (12+ rounds)
- JWT tokens with short expiration (15 minutes access, 7 days refresh)
- Input validation with Joi
- SQL injection prevention with parameterized queries
- XSS protection with helmet.js
- Rate limiting on all endpoints

### Privacy Compliance
- Data encryption at rest
- Audit logging for data access
- User data export/deletion capabilities
- Cookie consent management
- GDPR compliance features

This architecture provides a solid foundation for scaling and can handle the complexity of an AI-powered job recommendation system.

# Initialize project
npm init -y

# Install dependencies
npm install express pg redis bcryptjs jsonwebtoken joi helmet cors express-rate-limit winston dotenv cookie-parser express-async-errors

# Install dev dependencies
npm install --save-dev nodemon jest supertest

# Create directory structure
mkdir -p src/{config,controllers,services,middleware,routes,utils,models,workers} logs

# Start development server
npm run dev

