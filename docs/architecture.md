                    ┌──────────────────────┐
                    │      Next.js App      │
                    │  Admin + Learner UI   │
                    └──────────┬───────────┘
                               │
                    Server Actions / API
                               │
             ┌─────────────────┼─────────────────┐
             │                 │                 │
             ▼                 ▼                 ▼
       Domain Layer      AI Integrations    Background Jobs
             │                 │                 │
             │          ┌──────┴──────┐          │
             │          │             │          │
             │       Anthropic     Pinecone    Cron
             │          │             │          │
             │          └──────┬──────┘          │
             │                 │                 │
             └─────────────────┼─────────────────┘
                               ▼
                     ┌──────────────────┐
                     │ Supabase/Postgres│
                     │ Auth + RLS + DB  │
                     │ + Private Storage│
                     └──────────────────┘