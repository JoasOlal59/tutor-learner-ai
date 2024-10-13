# tutor-learner-ai
tutor-learner-ai
ai-edu-tools/
├── components/
│   ├── DocumentUploader.tsx
│   ├── LearnerInteraction.tsx
│   └── Layout.tsx
├── pages/
│   ├── api/
│   │   ├── documents/
│   │   │   └── upload.ts
│   │   ├── interact.ts
│   │   ├── feedback.ts
│   │   └── saveInteraction.ts
│   ├── _app.tsx
│   └── index.tsx
├── utils/
│   ├── auth.ts
│   ├── cache.ts
│   ├── educationalContent.ts
│   ├── rateLimit.ts
│   └── vectorSearch.ts
├── lib/
│   └── prisma.ts
├── prisma/
│   └── schema.prisma
├── .env
├── next.config.js
├── package.json
└── tsconfig.json
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id           Int           @id @default(autoincrement())
  email        String        @unique
  password     String
  role         String        @default("student")
  documents    Document[]
  interactions Interaction[]
}

model Document {
  id        Int                @id @default(autoincrement())
  userId    Int
  user      User               @relation(fields: [userId], references: [id])
  title     String
  content   String             @db.Text
  mimeType  String
  createdAt DateTime           @default(now())
  updatedAt DateTime           @updatedAt
  embedding DocumentEmbedding?
  sources   Source[]
}

model DocumentEmbedding {
  id         Int      @id @default(autoincrement())
  documentId Int      @unique
  document   Document @relation(fields: [documentId], references: [id])
  embedding  Float[]
}

model Interaction {
  id        Int      @id @default(autoincrement())
  userId    Int
  user      User     @relation(fields: [userId], references: [id])
  query     String
  answer    String   @db.Text
  rating    Int?
  saved     Boolean  @default(false)
  createdAt DateTime @default(now())
  sources   Source[]
}

model Source {
  id            Int          @id @default(autoincrement())
  interactionId Int
  interaction   Interaction  @relation(fields: [interactionId], references: [id])
  type          SourceType
  documentId    Int?
  document      Document?    @relation(fields: [documentId], references: [id])
  title         String
  url           String?
}

enum SourceType {
  DOCUMENT
  WEB
}
import { PrismaClient } from '@prisma/client'

declare global {
  var prisma: PrismaClient | undefined
}

export const prisma = global.prisma || new PrismaClient()

if (process.env.NODE_ENV !== 'production') global.prisma = prisma

export default prisma
import { NextApiRequest, NextApiResponse } from 'next'
import jwt from 'jsonwebtoken'

export interface AuthenticatedRequest extends NextApiRequest {
  user?: {
    id: number;
    email: string;
    role: string;
  };
}

export function authenticateToken(handler: (req: AuthenticatedRequest, res: NextApiResponse) => Promise<void>) {
  return async (req: AuthenticatedRequest, res: NextApiResponse) => {
    const authHeader = req.headers['authorization']
    const token = authHeader && authHeader.split(' ')[1]

    if (!token) {
      return res.status(401).json({ error: 'Authentication token required' })
    }

    try {
      const user = jwt.verify(token, process.env.JWT_SECRET!) as { id: number; email: string; role: string }
      req.user = user
      return handler(req, res)
    } catch (error) {
      return res.status(403).json({ error: 'Invalid token' })
    }
  }
}
import NodeCache from 'node-cache'

const cache = new NodeCache({ stdTTL: 600 }) // 10 minutes default TTL

export function cacheMiddleware(key: string, ttl?: number) {
  return async (req: any, res: any, next: () => void) => {
    const cachedResponse = cache.get(key)
    if (cachedResponse) {
      return res.json(cachedResponse)
    }
    res.sendResponse = res.json
    res.json = (body: any) => {
      cache.set(key, body, ttl)
      res.sendResponse(body)
    }
    next()
  }
}
import axios from 'axios'

const KHAN_ACADEMY_API = 'https://www.khanacademy.org/api/v1'
const WIKIPEDIA_API = 'https://en.wikipedia.org/w/api.php'

export async function fetchEducationalContent(query: string) {
  const [khanAcademyContent, wikipediaContent] = await Promise.all([
    fetchKhanAcademyContent(query),
    fetchWikipediaContent(query),
  ])

  return [...khanAcademyContent, ...wikipediaContent]
}

async function fetchKhanAcademyContent(query: string) {
  const response = await axios.get(`${KHAN_ACADEMY_API}/search`, {
    params: { query, limit: 3 },
  })
  return response.data.items.map((item: any) => ({
    title: item.title,
    content: item.description,
    source: 'Khan Academy',
    url: item.url,
  }))
}

async function fetchWikipediaContent(query: string) {
  const response = await axios.get(WIKIPEDIA_API, {
    params: {
      action: 'query',
      format: 'json',
      prop: 'extracts',
      exintro: true,
      explaintext: true,
      titles: query,
    },
  })
  const page = Object.values(response.data.query.pages)[0] as any
  return [{
    title: page.title,
    content: page.extract,
    source: 'Wikipedia',
    url: `https://en.wikipedia.org/?curid=${page.pageid}`,
  }]
}
import rateLimit from 'express-rate-limit'
import slowDown from 'express-slow-down'

export const applyMiddleware = (handler: any) => {
  return rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100 // limit each IP to 100 requests per windowMs
  })(slowDown({
    windowMs: 15 * 60 * 1000, // 15 minutes
    delayAfter: 50, // allow 50 requests per 15 minutes, then...
    delayMs: 500 // begin adding 500ms of delay per request above 50
  })(handler))
}
import { OpenAIApi, Configuration } from 'openai'
import prisma from '../lib/prisma'

const openai = new OpenAIApi(new Configuration({ apiKey: process.env.OPENAI_API_KEY }))

async function getEmbedding(text: string) {
  const response = await openai.createEmbedding({
    model: "text-embedding-ada-002",
    input: text,
  })
  return response.data.data[0].embedding
}

export async function indexDocument(documentId: number, content: string) {
  const embedding = await getEmbedding(content)
  await prisma.documentEmbedding.create({
    data: {
      documentId,
      embedding,
    },
  })
}

export async function searchDocuments(query: string, userId: number) {
  const queryEmbedding = await getEmbedding(query)
  
  const results = await prisma.$queryRaw`
    SELECT d.id, d.title, d.content,
           1 - (e.embedding <=> ${queryEmbedding}::vector) as similarity
    FROM "Document" d
    JOIN "DocumentEmbedding" e ON d.id = e."documentId"
    WHERE d."userId" = ${userId}
    ORDER BY similarity DESC
    LIMIT 5
  `
  
  return results
}
import { NextApiRequest, NextApiResponse } from 'next'
import formidable from 'formidable-serverless'
import fs from 'fs'
import { authenticateToken, AuthenticatedRequest } from '../../../utils/auth'
import prisma from '../../../lib/prisma'
import { indexDocument } from '../../../utils/vectorSearch'

export const config = {
  api: {
    bodyParser: false,
  },
}

const handler = async (req: AuthenticatedRequest, res: NextApiResponse) => {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method not allowed' })
  }

  const form = new formidable.IncomingForm()
  form.parse(req, async (err, fields, files) => {
    if (err) {
      return res.status(500).json({ error: 'Error parsing form data' })
    }

    const file = files.document as formidable.File
    if (!file) {
      return res.status(400).json({ error: 'No file uploaded' })
    }

    try {
      const content = fs.readFileSync(file.path, 'utf8')
      const document = await prisma.document.create({
        data: {
          userId: req.user!.id,
          title: fields.title as string,
          content,
          mimeType: file.type,
        },
      })

      await indexDocument(document.id, content)

      res.status(200).json({ message: 'Document uploaded and processed successfully', document })
    } catch (error) {
      console.error('Error processing document:', error)
      res.status(500).json({ error: 'Error processing document' })
    } finally {
      fs.unlinkSync(file.path)
    }
  })
}

export default authenticateToken(handler)
import { NextApiResponse } from 'next'
import { Configuration, OpenAIApi } from 'openai'
import { authenticateToken, AuthenticatedRequest } from '../../utils/auth'
import { searchDocuments } from '../../utils/vectorSearch'
import { fetchEducationalContent } from '../../utils/educationalContent'
import { applyMiddleware } from '../../utils/rateLimit'
import { cacheMiddleware } from '../../utils/cache'
import prisma from '../../lib/prisma'

const openai = new OpenAIApi(new Configuration({ apiKey: process.env.OPENAI_API_KEY }))

const handler = async (req: AuthenticatedRequest, res: NextApiResponse) => {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method not allowed' })
  }

  const { query } = req.body

  try {
    const relevantDocs = await searchDocuments(query, req.user!.id)
    const webContent = await fetchEducationalContent(query)

    const context = [
      ...relevantDocs.map((doc: any) => doc.content),
      ...webContent.map(content => content.content)
    ].join('\n\n')

    const completion = await openai.createCompletion({
      model: "text-davinci-002", // Replace with your fine-tuned model ID when available
      prompt: `Context: ${context}\n\nQuestion: ${query}\n\nAnswer:`,
      max_tokens: 150,
      n: 1,
      stop: null,
      temperature: 0.7,
    })

    const answer = completion.data.choices[0].text.trim()

    const interaction = await prisma.interaction.create({
      data: {
        userId: req.user!.id,
        query,
        answer,
        sources: {
          create: [
            ...relevantDocs.map((doc: any) => ({
              type: 'DOCUMENT',
              documentId: doc.id,
              title: doc.title,
            })),
            ...webContent.map(content => ({
              type: 'WEB',
              title: content.title,
              url: content.url,
            })),
          ],
        },
      },
      include: { sources: true },
    })

    res.status(200).json({ answer, interactionId: interaction.id, sources: interaction.sources })
  } catch (error) {
    console.error('Error processing interaction:', error)
    res.status(500).json({ error: 'Error processing interaction' })
  }
}

export default authenticateToken(applyMiddleware(cacheMiddleware('interaction', 300)(handler)))
import { NextApiResponse } from 'next'
import { authenticateToken, AuthenticatedRequest } from '../../utils/auth'
import prisma from '../../lib/prisma'

const handler = async (req: AuthenticatedRequest, res: NextApiResponse) => {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method not allowed' })
  }

  const { interactionId, rating } = req.body

  try {
    await prisma.interaction.update({
      where: { id: interactionId },
      data: { rating },
    })

    res.status(200).json({ message: 'Feedback recorded successfully' })
  } catch (error) {
    console.error('Error recording feedback:', error)
    res.status(500).json({ error: 'Error recording feedback' })
  }
}

export default authenticateToken(handler)
import { NextApiResponse } from 'next'
import { authenticateToken, AuthenticatedRequest } from '../../utils/auth'
import prisma from '../../lib/prisma'

const handler = async (req: AuthenticatedRequest, res: NextApiResponse) => {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method not allowed' })
  }

  const { interactionId } = req.body

  try {
    await prisma.interaction.update({
      where: { id: interactionId },
      data: { saved: true },
    })

    res.status(200).json({ message: 'Interaction saved successfully' })
  } catch (error) {
    console.error('Error saving interaction:', error)
    res.status(500).json({ error: 'Error saving interaction' })
  }
}

export default authenticateToken(handler)
import React, { useState } from 'react'
import axios from 'axios'

export default function DocumentUploader() {
  const [file, setFile] = useState<File | null>(null)
  const [title, setTitle] = useState('')
  const [uploading, setUploading] = useState(false)
  const [error, setError] = useState('')

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>)
  import React, { useState } from 'react'
import axios from 'axios'

export default function DocumentUploader() {
  const [file, setFile] = useState<File | null>(null)
  const [title, setTitle] = useState('')
  const [uploading, setUploading] = useState(false)
  const [error, setError] = useState('')

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.files) {
      setFile(e.target.files[0])
    }
  }

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    if (!file) return

    setUploading(true)
    setError('')

    const formData = new FormData()
    formData.append('document', file)
    formData.append('title', title)

    try {
      const response = await axios.post('/api/documents/upload', formData, {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      })
      console.log('Document uploaded:', response.data)
      setFile(null)
      setTitle('')
    } catch (error) {
      console.error('Error uploading document:', error)
      setError('Error uploading document. Please try again.')
    } finally {
      setUploading(false)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="Document title"
        required
      />
      <input type="file" onChange={handleFileChange} required />
      <button type="submit" disabled={uploading}>
        {uploading ? 'Uploading...' : 'Upload Document'}
      </button>
      {error && <p style={{ color: 'red' }}>{error}</p>}
    </form>
  )
}
import React, { useState } from 'react'
import axios from 'axios'

interface Source {
  id: number
  type: 'DOCUMENT' | 'WEB'
  title: string
  url?: string
}

export default function LearnerInteraction() {
  const [query, setQuery] = useState('')
  const [answer, setAnswer] = useState('')
  const [sources, setSources] = useState<Source[]>([])
  const [interactionId, setInteractionId] = useState<number | null>(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState('')

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setLoading(true)
    setError('')

    try {
      const response = await axios.post('/api/interact', { query })
      setAnswer(response.data.answer)
      setSources(response.data.sources)
      setInteractionId(response.data.interactionId)
    } catch (error) {
      console.error('Error interacting:', error)
      setError('Error processing your question. Please try again.')
    } finally {
      setLoading(false)
    }
  }

  const handleFeedback = async (rating: number) => {
    try {
      await axios.post('/api/feedback', { interactionId, rating })
      alert('Thank you for your feedback!')
    } catch (error) {
      console.error('Error submitting feedback:', error)
      alert('Error submitting feedback. Please try again.')
    }
  }

  const handleSave = async () => {
    try {
      await axios.post('/api/saveInteraction', { interactionId })
      alert('Interaction saved successfully!')
    } catch (error) {
      console.error('Error saving interaction:', error)
      alert('Error saving interaction. Please try again.')
    }
  }

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Ask a question"
          required
        />
        <button type="submit" disabled={loading}>
          {loading ? 'Processing...' : 'Ask'}
        </button>
      </form>
      {error && <p style={{ color: 'red' }}>{error}</p>}
      {answer && (
        <div>
          <h3>Answer:</h3>
          <p>{answer}</p>
          <h4>Sources:</h4>
          <ul>
            {sources.map((source) => (
              <li key={source.id}>
                {source.type === 'WEB' && source.url ? (
                  <a href={source.url} target="_blank" rel="noopener noreferrer">
                    {source.title}
                  </a>
                ) : (
                  source.title
                )}
              </li>
            ))}
          </ul>
          <div>
            <button onClick={() => handleFeedback(1)}>👍</button>
            <button onClick={() => handleFeedback(-1)}>👎</button>
            <button onClick={handleSave}>Save</button>
          </div>
        </div>
      )}
    </div>
  )
}
import React from 'react'
import Head from 'next/head'

interface LayoutProps {
  children: React.ReactNode
}

export default function Layout({ children }: LayoutProps) {
  return (
    <div>
      <Head>
        <title>AI Education Tools</title>
        <meta name="description" content="AI-powered education tools" />
        <link rel="icon" href="/favicon.ico" />
      </Head>

      <main>
        <header>
          <h1>AI Education Tools</h1>
        </header>
        {children}
      </main>

      <footer>
        <p>&copy; 2023 AI Education Tools</p>
      </footer>
    </div>
  )
}
import type { AppProps } from 'next/app'
import Layout from '../components/Layout'

function MyApp({ Component, pageProps }: AppProps) {
  return (
    <Layout>
      <Component {...pageProps} />
    </Layout>
  )
}

export default MyApp
import DocumentUploader from '../components/DocumentUploader'
import LearnerInteraction from '../components/LearnerInteraction'

export default function Home() {
  return (
    <div>
      <h2>Upload Documents</h2>
      <DocumentUploader />
      <h2>Ask Questions</h2>
      <LearnerInteraction />
    </div>
  )
}
module.exports = {
    reactStrictMode: true,
  }
  {
    "name": "ai-edu-tools",
    "version": "1.0.0",
    "private": true,
    "scripts": {
      "dev": "next dev",
      "build": "prisma generate && next build",
      "start": "next start",
      "lint": "next lint",
      "prisma:generate": "prisma generate",
      "prisma:migrate": "prisma migrate deploy",
      "postinstall": "prisma generate"
    },
    "dependencies": {
      "@prisma/client": "^4.14.0",
      "axios": "^1.4.0",
      "express-rate-limit": "^6.7.0",
      "express-slow-down": "^1.6.0",
      "formidable": "^2.1.1",
      "jsonwebtoken": "^9.0.0",
      "next": "^13.4.2",
      "node-cache": "^5.1.2",
      "openai": "^3.2.1",
      "react": "18.2.0",
      "react-dom": "18.2.0"
    },
    "devDependencies": {
      "@types/formidable": "^2.0.5",
      "@types/jsonwebtoken": "^9.0.2",
      "@types/node": "^20.1.3",
      "@types/react": "^18.2.6",
      "@types/react-dom": "^18.2.4",
      "eslint": "^8.40.0",
      "eslint-config-next": "^13.4.2",
      "prisma": "^4.14.0",
      "typescript": "^5.0.4"
    }
  }
  DATABASE_URL="postgresql://username:password@localhost:5432/ai_edu_tools?schema=public"
  
JWT_SECRET="your-jwt-secret"

OPENAI_API_KEY="your-openai-api-key"
