# Autonomous AI HR Decision Agent

> AI-Powered Human Resource Management System for Indian SMEs

[![AWS AI for Bharat Hackathon 2025](https://img.shields.io/badge/AWS-AI%20for%20Bharat%202025-orange)](https://github.com/03-lovepreetSingh/HRMS)
[![Team INVICTUS](https://img.shields.io/badge/Team-INVICTUS-blue)](https://github.com/03-lovepreetSingh/HRMS)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

## 🎯 Overview

The Autonomous AI HR Decision Agent is an intelligent HRMS that automates 70% of routine HR decisions using Large Language Models (Claude Sonnet 4.5, GPT-4). Built specifically for Indian SMEs with 50-500 employees, it reduces HR query response time from 24-48 hours to under 5 seconds.

### Key Features

- 🤖 **AI-Powered Chat Interface**: Natural language interaction for all HR queries
- ⚡ **Autonomous Decision-Making**: Instant approval/rejection of routine requests
- 📚 **Policy-Aware Processing**: Context-aware decisions based on company policies
- 🔄 **Smart Escalation**: Automatic routing of complex cases to human HR
- 📊 **Complete Audit Trail**: Full transparency and compliance tracking
- 👥 **Multi-Role Support**: Tailored experiences for Admin, HR Manager, Manager, and Employee

## 🏗️ Architecture

### Technology Stack

**Frontend**:
- Next.js 14 (App Router)
- React 18
- TypeScript
- Tailwind CSS
- shadcn/ui

**Backend**:
- Node.js
- Express.js
- TypeScript
- Drizzle ORM

**Database**:
- PostgreSQL 14+
- pgvector extension

**AI/LLM**:
- Claude Sonnet 4.5 (Primary)
- GPT-4 (Fallback)
- LangChain
- OpenAI Embeddings

**Infrastructure**:
- AWS (EC2, RDS, S3)
- Docker
- WebSocket (Real-time updates)

### System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    User Layer                                │
│  Employee  │  Manager  │  HR Manager  │  Admin              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Frontend (Next.js 14)                           │
│  Chat Interface  │  Dashboards  │  Admin Panel              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│           Backend Services (Node.js)                         │
│  AI Decision Engine  │  Leave Service  │  Policy Service    │
└─────────────────────────────────────────────────────────────┘
                            │
                    ┌───────┴───────┐
                    │               │
            ┌───────▼─────┐  ┌─────▼──────┐
            │   Claude    │  │   GPT-4    │
            │ Sonnet 4.5  │  │ (Fallback) │
            └─────────────┘  └────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│         PostgreSQL + pgvector                                │
│  Relational Data  │  Vector Embeddings  │  Audit Logs       │
└─────────────────────────────────────────────────────────────┘
```

## 📁 Project Structure

```
hrms/
├── apps/
│   └── web/                    # Next.js frontend application
│       ├── app/               # App Router pages
│       │   ├── (auth)/        # Authentication pages
│       │   ├── (dashboard)/   # Dashboard pages
│       │   └── api/           # API routes
│       ├── components/        # React components
│       │   ├── chat/          # Chat interface
│       │   ├── dashboard/     # Dashboard components
│       │   └── ui/            # shadcn/ui components
│       └── lib/               # Utility functions
│
├── packages/
│   ├── database/              # Database schema and migrations
│   │   ├── schema/            # Drizzle schema definitions
│   │   └── migrations/        # Database migrations
│   ├── ai/                    # AI/LLM integration
│   │   ├── decision-engine/   # Decision engine logic
│   │   └── langchain/         # LangChain setup
│   ├── types/                 # Shared TypeScript types
│   └── config/                # Shared configuration
│
├── .kiro/
│   └── specs/
│       └── autonomous-ai-hr-agent/
│           ├── requirements.md    # Software Requirements Specification
│           ├── design.md          # Design Document
│           └── tasks.md           # Implementation Tasks (TBD)
│
├── turbo.json                 # Turborepo configuration
└── package.json               # Root package.json
```

## 📋 Documentation

### Specification Documents

All project specifications are located in `.kiro/specs/autonomous-ai-hr-agent/`:

1. **[requirements.md](.kiro/specs/autonomous-ai-hr-agent/requirements.md)** - Comprehensive Software Requirements Specification (SRS)
   - Executive Summary
   - Business Requirements (5 objectives)
   - Functional Requirements (60+ requirements across 12 modules)
   - Non-Functional Requirements (19 requirements across 8 categories)
   - System Constraints
   - Data Models
   - Implementation Phases
   - Risk Analysis

2. **[design.md](.kiro/specs/autonomous-ai-hr-agent/design.md)** - Detailed Design Document
   - System Architecture
   - Component Interfaces
   - Data Models
   - Correctness Properties (30 properties)
   - Error Handling
   - Testing Strategy

3. **tasks.md** - Implementation Plan (Coming Soon)

### Key Modules

#### User Management (UM)
- Authentication with JWT
- Role-Based Access Control (RBAC)
- User Profile Management

#### AI Chat Interface (AI)
- Natural Language Processing
- Conversational Context Management
- Multi-Modal Response Generation
- Real-Time Typing Indicators

#### Leave Management (LM)
- Leave Request Submission via Chat
- Autonomous Leave Approval
- Smart Escalation
- Leave Balance Tracking

#### Policy Management (PM)
- Policy Document Storage
- Semantic Search with pgvector
- Policy Versioning
- Policy Approval Workflow

#### AI Decision Engine (DE)
- Context Assembly
- LLM-Based Decision Making
- Confidence-Based Routing
- Validation Rules
- Learning from Feedback

#### Escalation Management (EM)
- Automatic Routing
- Queue Management
- SLA Tracking
- Collaboration Features

#### Audit & Compliance (AC)
- Comprehensive Logging
- Audit Trail Search
- Compliance Reporting
- Decision Explainability

## 🚀 Getting Started

### Prerequisites

- Node.js 18+
- PostgreSQL 14+ with pgvector extension
- Docker (optional)
- AWS Account
- Anthropic API Key (Claude)
- OpenAI API Key (GPT-4)

### Installation

```bash
# Clone the repository
git clone https://github.com/03-lovepreetSingh/HRMS.git
cd HRMS

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env
# Edit .env with your API keys and database credentials

# Set up database
npm run db:push
npm run db:seed

# Start development server
npm run dev
```

### Environment Variables

```env
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/hrms

# LLM APIs
ANTHROPIC_API_KEY=your_claude_api_key
OPENAI_API_KEY=your_openai_api_key

# Authentication
JWT_SECRET=your_jwt_secret
JWT_EXPIRES_IN=24h

# AWS
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_S3_BUCKET=your_bucket_name

# Email
EMAIL_SERVICE=ses
EMAIL_FROM=noreply@yourcompany.com
```

## 🧪 Testing

### Run Tests

```bash
# Run all tests
npm test

# Run unit tests
npm run test:unit

# Run property-based tests
npm run test:property

# Run integration tests
npm run test:integration

# Run with coverage
npm run test:coverage
```

### Testing Strategy

- **Unit Tests** (40%): Specific examples and edge cases
- **Property-Based Tests** (40%): Universal properties with 100+ iterations
- **Integration Tests** (15%): End-to-end flows
- **Manual Tests** (5%): UI/UX validation

**Coverage Target**: > 80%

## 📊 Key Metrics

### Performance Targets

- AI Response Time: < 5 seconds (95th percentile)
- Page Load Time: < 2 seconds
- API Response Time: < 500ms
- System Uptime: 99.5%

### Business Metrics

- Automation Rate: 70% of HR decisions
- Decision Accuracy: 95%
- Employee Satisfaction: 90%
- Cost Reduction: 40%

## 🎯 Implementation Phases

### Phase 1: Hackathon MVP (Weeks 1-4) ✅

- User authentication and RBAC
- AI chat interface
- Leave request and auto-approval
- Policy upload and semantic search
- AI decision engine
- Basic escalation
- Audit logging

### Phase 2: Post-Hackathon Enhancement (Weeks 5-12)

- Attendance management
- Task management
- Advanced analytics
- Enhanced escalation
- Performance optimization
- Security hardening

### Phase 3: Scale and Optimize (Weeks 13-24)

- Multi-tenancy
- Mobile applications
- Multi-language support
- Advanced AI features
- Integration with existing HRMS
- White-labeling

## 🔒 Security

- All data encrypted at rest (AES-256) and in transit (TLS 1.3)
- JWT-based authentication
- Role-Based Access Control (RBAC)
- Rate limiting (100 requests/minute per user)
- Input validation and sanitization
- SQL injection prevention
- XSS prevention
- Comprehensive audit logging

## 📈 Monitoring

- Application Performance Monitoring (APM)
- Real-time error tracking
- LLM API usage and cost monitoring
- Database performance monitoring
- User activity analytics
- Automated alerting

## 🤝 Contributing

We welcome contributions! Please see our [Contributing Guidelines](CONTRIBUTING.md) for details.

### Development Workflow

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 👥 Team INVICTUS

- **Team Leader**: Lovepreet Singh
- **Project**: AWS AI for Bharat Hackathon 2025
- **Repository**: https://github.com/03-lovepreetSingh/HRMS

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- AWS AI for Bharat Hackathon 2025
- Anthropic (Claude API)
- OpenAI (GPT-4 API)
- pgvector community
- Next.js team
- shadcn/ui

## 📞 Support

For questions or support:
- Create an issue in this repository
- Email: support@yourcompany.com
- Documentation: [View Specs](.kiro/specs/autonomous-ai-hr-agent/)

---

**Built with ❤️ by Team INVICTUS for AWS AI for Bharat Hackathon 2025**
