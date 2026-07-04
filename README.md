

Deel 1 — Kennis (12 doelen)
1. Het verschil tussen client en server componenten
In Next.js 16 zijn componenten standaard Server Components. Ze draaien op de server, hebben geen JavaScript-bundel voor de browser, en kunnen rechtstreeks data ophalen (bijv. uit een database).

Server Component — draait op de server:

tsx

// app/dashboard/page.tsx
// Geen 'use client' = server component (standaard)
export default async function DashboardPage() {
  const posts = await db.post.findMany()

  return (
    <div>
      <h1>Dashboard</h1>
      {posts.map((post) => (
        <p key={post.id}>{post.title}</p>
      ))}
    </div>
  )
}
Client Component — draait in de browser:

tsx

// app/components/LikeButton.tsx
'use client' // <-- dit maakt het een client component

import { useState } from 'react'

export default function LikeButton() {
  const [likes, setLikes] = useState(0)

  return <button onClick={() => setLikes(likes + 1)}>❤️ {likes}</button>
}
Wanneer gebruik je wat?

Server Component	Client Component
Data ophalen uit database	Interactie (klikken, typen)
Gevoelige logica (API keys)	useState, useEffect nodig
Geen interactiviteit nodig	Browser API's (localStorage)
Onthoud: 'use client' bovenaan het bestand maakt het een client component. Zonder die directive is het automatisch een server component.

2. Het verschil tussen authenticatie en autorisatie
Dit zijn twee verschillende stappen in beveiliging:

Authenticatie = Wie ben je? De gebruiker bewijst zijn identiteit, bijvoorbeeld door in te loggen met e-mail en wachtwoord.

Autorisatie = Wat mag je doen? Na het inloggen wordt gecontroleerd welke rechten de gebruiker heeft. Mag deze gebruiker de admin-pagina zien?

Gebruiker logt in (authenticatie)
        ↓
Systeem checkt: is dit een admin? (autorisatie)
        ↓
   Ja → Toon admin dashboard
   Nee → Toon gewone gebruikerspagina
Ezelsbruggetje: Authenticatie = je ID-kaart laten zien bij de deur. Autorisatie = of je ook naar de VIP-ruimte mag.

3. Wat is een JSON Web Token (JWT)?
Een JWT is een gecodeerde string die informatie bevat over een gebruiker. Het wordt gebruikt voor stateless sessies — de server hoeft geen sessie op te slaan, want alle informatie zit in het token zelf.

Een JWT bestaat uit drie delen, gescheiden door punten:

header.payload.signature
Header: het algoritme (bijv. HS256)
Payload: de data (bijv. userId, role, vervaldatum)
Signature: een handtekening waarmee je controleert of het token niet is aangepast
Voorbeeld payload:

json

{
  "userId": "42",
  "role": "admin",
  "exp": 1735689600
}
In Next.js kun je met de jose-library een JWT aanmaken en verifiëren:

ts

import { SignJWT, jwtVerify } from 'jose'

const secret = new TextEncoder().encode(process.env.SESSION_SECRET)

// Token aanmaken
const token = await new SignJWT({ userId: '42', role: 'admin' })
  .setProtectedHeader({ alg: 'HS256' })
  .setExpirationTime('7d')
  .sign(secret)

// Token verifiëren
const { payload } = await jwtVerify(token, secret)
console.log(payload.userId) // '42'
4. Wat is Role Based Access Control (RBAC)?
Bij RBAC geef je gebruikers een rol (bijv. ADMIN, USER, DOCENT), en op basis van die rol bepaal je wat ze mogen doen of zien.

tsx

export default async function AdminPanel() {
  const session = await verifySession()

  if (session.role !== 'ADMIN') {
    redirect('/unauthorized')
  }

  return (
    <div>
      <h1>Admin Panel</h1>
      <button>Gebruiker verwijderen</button>
    </div>
  )
}
In je Prisma-schema definieer je rollen met een enum:

prisma

enum Role {
  USER
  ADMIN
  DOCENT
}

model User {
  id       Int    @id @default(autoincrement())
  email    String @unique
  password String
  role     Role   @default(USER)
}
5. Wat is een ORM?
ORM staat voor Object-Relational Mapping. Het is een laag tussen je code en de database waarmee je tabellen benadert als objecten in je programmeertaal, in plaats van raw SQL te schrijven.

Zonder ORM (raw SQL):
  SELECT * FROM users WHERE email = 'bas@test.nl'

Met ORM (Prisma):
  prisma.user.findUnique({ where: { email: 'bas@test.nl' } })
Voordelen: type-safety, makkelijker te lezen, automatische migraties, minder kans op SQL-injectie.

6. Wat is Prisma?
Prisma is een ORM voor Node.js en TypeScript. In Prisma 7 is de architectuur vernieuwd: er is een nieuw configuratiebestand (prisma.config.ts), de client is Rust-free en werkt met ESM, en je moet een database adapter meegeven.

De belangrijkste onderdelen:

schema.prisma — hier definieer je je modellen (tabellen)
prisma.config.ts — configuratie: database-URL, seed-script, migraties (nieuw in Prisma 7)
Prisma Client — de gegenereerde client waarmee je queries uitvoert
Prisma Migrate — beheert je databasemigraties
Voorbeeld prisma.config.ts (Prisma 7):

ts

import 'dotenv/config'
import { defineConfig, env } from 'prisma/config'

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: {
    path: 'prisma/migrations',
    seed: 'tsx prisma/seed.ts',
  },
  datasource: {
    url: env('DATABASE_URL'),
  },
})
Voorbeeld schema.prisma (Prisma 7):

prisma

generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "mysql"
}

model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  name  String
}
Let op Prisma 7 veranderingen:

provider is nu "prisma-client" (was "prisma-client-js")
De database url staat niet meer in schema.prisma maar in prisma.config.ts
output is verplicht — genereer de client in je src/-map
npx prisma generate moet je nu altijd handmatig draaien
7. One-to-Many vs Many-to-Many relaties
One-to-Many (1:N) — Eén record heeft meerdere gerelateerde records.

Voorbeeld: Eén gebruiker heeft meerdere posts.

prisma

model User {
  id    Int    @id @default(autoincrement())
  name  String
  posts Post[]   // een user heeft VEEL posts
}

model Post {
  id     Int    @id @default(autoincrement())
  title  String
  user   User   @relation(fields: [userId], references: [id])
  userId Int    // foreign key naar User
}
Many-to-Many (M:N) — Meerdere records aan beide kanten.

Voorbeeld: Studenten volgen meerdere cursussen, cursussen hebben meerdere studenten.

prisma

model Student {
  id      Int      @id @default(autoincrement())
  name    String
  courses Course[]  // een student volgt VEEL cursussen
}

model Course {
  id       Int       @id @default(autoincrement())
  name     String
  students Student[] // een cursus heeft VEEL studenten
}
8. Wat betekent seeden?
Seeden is het vullen van je database met testdata of startdata. Dit doe je met een seed-script dat je uitvoert met npx prisma db seed.

In Prisma 7 configureer je het seed-script in prisma.config.ts:

ts

export default defineConfig({
  // ...
  migrations: {
    path: 'prisma/migrations',
    seed: 'tsx prisma/seed.ts', // <-- seed-commando
  },
})
Voorbeeld prisma/seed.ts:

ts

import 'dotenv/config'
import { PrismaClient } from '../src/generated/prisma/client'
import { PrismaMariaDb } from '@prisma/adapter-mariadb'
import mariadb from 'mariadb'

const pool = mariadb.createPool({ uri: process.env.DATABASE_URL })
const adapter = new PrismaMariaDb(pool)
const prisma = new PrismaClient({ adapter })

async function main() {
  await prisma.user.deleteMany() // opschonen

  await prisma.user.create({
    data: {
      email: 'admin@school.nl',
      name: 'Admin',
      role: 'ADMIN',
    },
  })

  console.log('Database is geseeded!')
}

main().finally(async () => {
  await prisma.$disconnect()
  await pool.end()
})
Uitvoeren:

bash

npx prisma db seed
Let op: In Prisma 7 wordt seeden niet meer automatisch uitgevoerd na migraties. Je moet het altijd expliciet aanroepen.

9. Expliciet vs impliciet M:N relatie
Impliciete M:N — Prisma maakt automatisch een koppeltabel aan. Je ziet deze tabel niet in je schema:

prisma

model Student {
  id      Int      @id @default(autoincrement())
  name    String
  courses Course[]
}

model Course {
  id       Int       @id @default(autoincrement())
  name     String
  students Student[]
}
Prisma maakt op de achtergrond een tabel _CourseToStudent aan. Je hebt geen controle over die tabel.

Expliciete M:N — Je maakt zelf de koppeltabel als model. Dit is nodig als je extra velden wilt opslaan op de relatie:

prisma

model Student {
  id          Int          @id @default(autoincrement())
  name        String
  enrollments Enrollment[]
}

model Course {
  id          Int          @id @default(autoincrement())
  name        String
  enrollments Enrollment[]
}

// De koppeltabel met extra velden
model Enrollment {
  id         Int      @id @default(autoincrement())
  student    Student  @relation(fields: [studentId], references: [id])
  studentId  Int
  course     Course   @relation(fields: [courseId], references: [id])
  courseId   Int
  grade      Float?          // extra veld: cijfer
  enrolledAt DateTime @default(now()) // extra veld: datum

  @@unique([studentId, courseId]) // voorkomt dubbele inschrijvingen
}
Vuistregel: Heb je extra data nodig op de relatie (cijfer, datum, status)? → Expliciet. Anders volstaat impliciet.

10. Wat is een koppeltabel?
Een koppeltabel (ook: join table, tussentabel) is een tabel die twee andere tabellen aan elkaar koppelt in een many-to-many relatie. De koppeltabel bevat de foreign keys van beide tabellen.

Student              Enrollment (koppeltabel)       Course
┌────┬────────┐     ┌───────────┬──────────┐     ┌────┬────────┐
│ id │ name   │     │ studentId │ courseId  │     │ id │ name   │
├────┼────────┤     ├───────────┼──────────┤     ├────┼────────┤
│  1 │ Yusuf  │────▶│     1     │    1     │◀────│  1 │ Web Dev│
│  2 │ Sara   │────▶│     1     │    2     │◀────│  2 │ Data   │
│    │        │     │     2     │    1     │     │    │        │
└────┴────────┘     └───────────┴──────────┘     └────┴────────┘
Yusuf volgt Web Dev én Data. Sara volgt Web Dev. Zonder de koppeltabel zou je deze relatie niet kunnen opslaan.

11. Wat doet revalidatePath?
revalidatePath is een Next.js-functie die de cache van een specifiek pad ongeldig maakt, zodat de pagina opnieuw wordt opgehaald met verse data.

Wanneer je data muteert (bijv. een post toevoegen) via een Server Action, moet je Next.js vertellen dat de oude gecachte pagina niet meer klopt:

ts

'use server'

import { revalidatePath } from 'next/cache'

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string

  await prisma.post.create({
    data: { title },
  })

  revalidatePath('/posts') // ← cache van /posts wordt ververst
}
Zonder revalidatePath zou de pagina oude data blijven tonen totdat de cache verloopt.

12. Wat zijn async functions?
Een async function is een functie die asynchroon werkt — dat betekent dat de functie kan wachten op taken die tijd kosten (zoals data ophalen uit een database of een API aanroepen) zonder de rest van je applicatie te blokkeren.

Je herkent ze aan twee keywords: async en await.

ts

// Zonder async — dit werkt NIET goed:
function getUser() {
  const user = prisma.user.findUnique({ where: { id: 1 } })
  console.log(user) // ❌ Promise { <pending> } — je hebt de data nog niet!
}

// Met async/await — dit werkt WEL:
async function getUser() {
  const user = await prisma.user.findUnique({ where: { id: 1 } })
  console.log(user) // ✅ { id: 1, name: 'Bas', email: 'bas@test.nl' }
}
Waarom is dit belangrijk in Next.js?

Server Components in Next.js mogen async zijn. Dat is wat ze zo krachtig maakt — je kunt direct in je component op de database wachten:

tsx

// Server Component — mag async zijn
export default async function UsersPage() {
  const users = await prisma.user.findMany() // wacht op database

  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>
}
Server Actions zijn ook altijd async:

ts

'use server'

export async function deleteUser(id: number) {
  await prisma.user.delete({ where: { id } }) // wacht tot verwijderd
  revalidatePath('/users')
}
Belangrijke regels:

await kan alleen binnen een async functie
Een async functie retourneert altijd een Promise
In Next.js: Server Components mogen async zijn, Client Components niet
Database-operaties, API-calls en cookies() zijn allemaal async
Deel 2 — Vaardigheden (10 doelen)
13. Een server component maken
Een server component is het standaardgedrag in Next.js 16 — je hoeft niets speciaals te doen. Ze kunnen async zijn en direct data ophalen:

tsx

// app/users/page.tsx — dit is een server component (standaard)
export default async function UsersPage() {
  const users = await prisma.user.findMany()

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name} — {user.email}</li>
      ))}
    </ul>
  )
}
14. Een client component maken
Voeg 'use client' toe bovenaan het bestand. Client components zijn nodig voor interactiviteit:

tsx

// app/components/SearchBar.tsx
'use client'

import { useState } from 'react'

export default function SearchBar() {
  const [query, setQuery] = useState('')

  return (
    <input
      type="text"
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Zoeken..."
    />
  )
}
Je kunt een client component gebruiken binnen een server component:

tsx

// app/page.tsx (server component)
import SearchBar from './components/SearchBar'

export default async function Page() {
  const data = await prisma.post.findMany()

  return (
    <div>
      <SearchBar /> {/* client component in een server component */}
      {data.map((post) => <p key={post.id}>{post.title}</p>)}
    </div>
  )
}
15. Een wachtwoord hashen
Je slaat wachtwoorden nooit op als platte tekst. Gebruik bcryptjs om een wachtwoord te hashen:

bash

npm install bcryptjs
npm install -D @types/bcryptjs
ts

import bcrypt from 'bcryptjs'

// Hashen (bij registratie)
const plainPassword = 'MijnVeiligWachtwoord123!'
const hashedPassword = await bcrypt.hash(plainPassword, 10)
// Resultaat: '$2b$10$xK3v...' — een onleesbare hash

// Opslaan in de database
await prisma.user.create({
  data: {
    email: 'student@school.nl',
    password: hashedPassword, // sla de HASH op, niet het wachtwoord
  },
})
Het getal 10 is het aantal salt rounds — hoe hoger, hoe veiliger maar ook langzamer.

16. Een wachtwoord vergelijken met een hash
Bij het inloggen vergelijk je het ingevoerde wachtwoord met de hash in de database:

ts

import bcrypt from 'bcryptjs'

export async function login(formData: FormData) {
  const email = formData.get('email') as string
  const password = formData.get('password') as string

  // Gebruiker ophalen
  const user = await prisma.user.findUnique({
    where: { email },
  })

  if (!user) {
    return { error: 'Gebruiker niet gevonden' }
  }

  // Wachtwoord vergelijken met opgeslagen hash
  const isValid = await bcrypt.compare(password, user.password)

  if (!isValid) {
    return { error: 'Onjuist wachtwoord' }
  }

  // Wachtwoord klopt → sessie aanmaken
  await createSession(user.id)
}
Belangrijk: bcrypt.compare() hasht het ingevoerde wachtwoord met dezelfde salt en vergelijkt het resultaat. Je kunt een hash nooit terug omzetten naar het originele wachtwoord.

17. Een Prisma schema aanpassen
Stel je hebt een bestaand schema en je wilt een role-veld toevoegen aan User:

Stap 1 — Schema aanpassen in prisma/schema.prisma:

prisma

enum Role {
  USER
  ADMIN
}

model User {
  id       Int    @id @default(autoincrement())
  email    String @unique
  name     String
  password String
  role     Role   @default(USER) // ← nieuw veld
  posts    Post[]
}
Stap 2 — Migratie aanmaken:

bash

npx prisma migrate dev --name add-role-to-user
Stap 3 — Client opnieuw genereren:

bash

npx prisma generate
Nu kun je role gebruiken in je queries:

ts

const admins = await prisma.user.findMany({
  where: { role: 'ADMIN' },
})
18. Een database seeden
Zie ook toetsdoel 8. Hier een volledig voorbeeld:

prisma/seed.ts

ts

import 'dotenv/config'
import { PrismaClient } from '../src/generated/prisma/client'
import { PrismaMariaDb } from '@prisma/adapter-mariadb'
import mariadb from 'mariadb'
import bcrypt from 'bcryptjs'

const pool = mariadb.createPool({ uri: process.env.DATABASE_URL })
const adapter = new PrismaMariaDb(pool)
const prisma = new PrismaClient({ adapter })

async function main() {
  // Opschonen (let op volgorde i.v.m. relaties!)
  await prisma.post.deleteMany()
  await prisma.user.deleteMany()

  // Admin aanmaken
  const admin = await prisma.user.create({
    data: {
      email: 'admin@school.nl',
      name: 'Admin',
      password: await bcrypt.hash('Admin123!', 10),
      role: 'ADMIN',
    },
  })

  // Posts aanmaken
  await prisma.post.createMany({
    data: [
      { title: 'Eerste post', content: 'Hallo wereld', userId: admin.id },
      { title: 'Tweede post', content: 'Nog een post', userId: admin.id },
    ],
  })

  console.log('Seeding voltooid!')
}

main().finally(async () => {
  await prisma.$disconnect()
  await pool.end()
})
19. Een Prisma schema maken volgens best practices
prisma

generator client {
  provider = "prisma-client"           // 1. Gebruik "prisma-client" (Prisma 7)
  output   = "../src/generated/prisma"  // 2. Output in je src-map
}

datasource db {
  provider = "mysql"               // 3. URL staat in prisma.config.ts
}

// 4. Gebruik enums voor vaste waarden
enum Role {
  USER
  ADMIN
}

// 5. Gebruik duidelijke modelnamen (enkelvoud, PascalCase)
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique              // 6. Unique waar nodig
  name      String
  password  String
  role      Role     @default(USER)       // 7. Standaardwaarden instellen
  posts     Post[]                        // 8. Relaties duidelijk benoemen
  createdAt DateTime @default(now())      // 9. Timestamps toevoegen
  updatedAt DateTime @updatedAt
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String
  published Boolean  @default(false)
  user      User     @relation(fields: [userId], references: [id])
  userId    Int                           // 10. Foreign key bij de "many"-kant
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
Samenvatting best practices:

Gebruik "prisma-client" als provider (Prisma 7)
Stel een output-pad in binnen je src/-map
Gebruik @unique op velden die uniek moeten zijn (email, slug)
Gebruik @default() voor standaardwaarden
Voeg createdAt en updatedAt toe aan je modellen
Gebruik enums voor velden met een beperkt aantal opties
Zet de foreign key (userId) altijd bij de "many"-kant van de relatie
Gebruik @@unique voor samengestelde unieke waarden
20. Server ↔ client logica uitleggen
De kern: server components draaien op de server en client components draaien in de browser. Ze werken samen, maar hebben elk hun eigen mogelijkheden.

┌─────────────── SERVER ───────────────┐     ┌────────── BROWSER ──────────┐
│                                      │     │                             │
│  Server Component                    │     │  Client Component           │
│  - Kan database benaderen            │────▶│  - Kan useState/useEffect   │
│  - Kan Server Actions aanroepen      │     │  - Kan onClick, onChange     │
│  - Kan secrets/env vars gebruiken    │     │  - Kan GEEN database direct  │
│  - Rendert HTML op de server         │     │  - Heeft JavaScript nodig   │
│                                      │     │                             │
│  Server Action ('use server')        │     │  Roept Server Actions aan   │
│  - Draait altijd op de server        │◀────│  - Via form action           │
│  - Kan data muteren                  │     │  - Via functie-aanroep       │
│  - Kan revalidatePath aanroepen      │     │                             │
└──────────────────────────────────────┘     └─────────────────────────────┘
Praktijkvoorbeeld — een pagina met een verwijderknop:

tsx

// app/posts/page.tsx — SERVER COMPONENT
import DeleteButton from './DeleteButton'

export default async function PostsPage() {
  // ✅ Dit draait op de server — database direct benaderen
  const posts = await prisma.post.findMany()

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>
          {post.title}
          <DeleteButton postId={post.id} /> {/* client component */}
        </li>
      ))}
    </ul>
  )
}
tsx

// app/posts/DeleteButton.tsx — CLIENT COMPONENT
'use client'

import { deletePost } from './actions'

export default function DeleteButton({ postId }: { postId: number }) {
  return (
    // ✅ onClick is interactie → client component nodig
    <button onClick={() => deletePost(postId)}>Verwijderen</button>
  )
}
ts

// app/posts/actions.ts — SERVER ACTION
'use server'

import { revalidatePath } from 'next/cache'

export async function deletePost(postId: number) {
  // ✅ Dit draait op de server — veilig database benaderen
  await prisma.post.delete({ where: { id: postId } })
  revalidatePath('/posts')
}
De flow: Browser (client component) → roept server action aan → server muteert data → revalidatePath ververst de pagina.

21. revalidatePath gebruiken
revalidatePath gebruik je in Server Actions na het muteren van data. Zo weet Next.js dat de gecachte pagina verouderd is en opnieuw gerenderd moet worden.

Voorbeeld 1 — Na het aanmaken van een record:

ts

'use server'

import { revalidatePath } from 'next/cache'

export async function createPost(formData: FormData) {
  await prisma.post.create({
    data: { title: formData.get('title') as string },
  })

  revalidatePath('/posts') // de /posts pagina wordt ververst
}
Voorbeeld 2 — Na het updaten van een specifiek record:

ts

'use server'

import { revalidatePath } from 'next/cache'

export async function updatePost(id: number, formData: FormData) {
  await prisma.post.update({
    where: { id },
    data: { title: formData.get('title') as string },
  })

  revalidatePath('/posts')       // lijst-pagina verversen
  revalidatePath(`/posts/${id}`) // detail-pagina ook verversen
}
Voorbeeld 3 — Gebruiken vanuit een formulier:

tsx

// Client component met een form
'use client'

import { createPost } from './actions'

export default function NewPostForm() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Titel..." />
      <button type="submit">Opslaan</button>
    </form>
  )
}
Onthoud: Zonder revalidatePath ziet de gebruiker de oude data totdat de cache vanzelf verloopt. Gebruik het altijd na een create, update of delete.

22. package.json scripts aanpassen
De scripts in package.json zijn snelkoppelingen voor commando's die je vaak gebruikt. Je voert ze uit met npm run <scriptnaam>.

Standaard Next.js 16 scripts:

json

{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint"
  }
}
Wat doen ze?

Script	Commando	Wat doet het?
npm run dev	next dev	Start de dev-server (met Turbopack)
npm run build	next build	Bouwt je app voor productie
npm run start	next start	Start de productie-server
npm run lint	eslint	Controleert je code op fouten
Scripts toevoegen voor Prisma:

json

{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint",
    "db:generate": "prisma generate",
    "db:migrate": "prisma migrate dev",
    "db:seed": "prisma db seed",
    "db:studio": "prisma studio",
    "db:reset": "prisma migrate reset"
  }
}
Nu kun je ze gebruiken:

bash

npm run db:migrate          # migratie draaien
npm run db:generate         # client opnieuw genereren
npm run db:seed             # database vullen met testdata
npm run db:studio           # visuele database viewer openen
npm run db:migrate -- --name add-role  # extra flags meegeven met --
Veelgemaakte toevoegingen:

json

{
  "scripts": {
    "db:setup": "prisma migrate dev && prisma db seed"
  }
}
Met && voer je meerdere commando's achter elkaar uit. Hier: eerst migreren, dan seeden.
