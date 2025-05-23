//------------------------
// DATABASE SCHEMA (shared/schema.ts)
//------------------------
import { pgTable, text, serial, integer, boolean, timestamp, pgEnum, foreignKey, unique } from "drizzle-orm/pg-core";
import { createInsertSchema } from "drizzle-zod";
import { z } from "zod";
import { relations } from "drizzle-orm";

// User schema
export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  username: text("username").notNull().unique(),
  password: text("password").notNull(),
});

export const insertUserSchema = createInsertSchema(users).pick({
  username: true,
  password: true,
});

export type InsertUser = z.infer<typeof insertUserSchema>;
export type User = typeof users.$inferSelect;

// Study set categories
export const studySetCategoryEnum = pgEnum('study_set_category', [
  'biology', 'computer_science', 'history', 'mathematics', 
  'physics', 'chemistry', 'literature', 'languages', 'other'
]);

// Study Sets
export const studySets = pgTable("study_sets", {
  id: serial("id").primaryKey(),
  title: text("title").notNull(),
  description: text("description"),
  category: studySetCategoryEnum("category").notNull(),
  userId: integer("user_id").notNull().references(() => users.id, { onDelete: 'cascade' }),
});

export const insertStudySetSchema = createInsertSchema(studySets).pick({
  title: true,
  description: true,
  category: true,
  userId: true,
});

export type InsertStudySet = z.infer<typeof insertStudySetSchema>;
export type StudySet = typeof studySets.$inferSelect;

// Flashcards
export const flashcards = pgTable("flashcards", {
  id: serial("id").primaryKey(),
  question: text("question").notNull(),
  answer: text("answer").notNull(),
  studySetId: integer("study_set_id").notNull().references(() => studySets.id, { onDelete: 'cascade' }),
  category: text("category"),
  timesReviewed: integer("times_reviewed").default(0),
  timesCorrect: integer("times_correct").default(0),
});

export const insertFlashcardSchema = createInsertSchema(flashcards).pick({
  question: true,
  answer: true,
  studySetId: true,
  category: true,
});

export type InsertFlashcard = z.infer<typeof insertFlashcardSchema>;
export type Flashcard = typeof flashcards.$inferSelect;

// Quiz schema
export const quizzes = pgTable("quizzes", {
  id: serial("id").primaryKey(),
  title: text("title").notNull(),
  studySetId: integer("study_set_id").notNull().references(() => studySets.id, { onDelete: 'cascade' }),
  createdAt: timestamp("created_at").defaultNow(),
});

// Quiz questions, notes, and study progress schemas follow the same pattern

// Define all relations
export const usersRelations = relations(users, ({ many }) => ({
  studySets: many(studySets),
  progress: many(studyProgress)
}));

export const studySetsRelations = relations(studySets, ({ one, many }) => ({
  user: one(users, {
    fields: [studySets.userId],
    references: [users.id]
  }),
  flashcards: many(flashcards),
  quizzes: many(quizzes),
  notes: many(notes),
  progress: many(studyProgress)
}));

// Other relationship definitions follow the same pattern

//------------------------
// DATABASE CONNECTION (server/db.ts)
//------------------------
import { Pool, neonConfig } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-serverless';
import ws from "ws";
import * as schema from "@shared/schema";

// Configure neon to use websockets
neonConfig.webSocketConstructor = ws;

if (!process.env.DATABASE_URL) {
  throw new Error(
    "DATABASE_URL must be set. Did you forget to provision a database?",
  );
}

export const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle({ client: pool, schema });

//------------------------
// STORAGE INTERFACE (server/storage.ts)
//------------------------
// Using DatabaseStorage implementation that interacts with PostgreSQL
export class DatabaseStorage implements IStorage {
  async getUser(id: number): Promise<User | undefined> {
    const [user] = await db.select().from(users).where(eq(users.id, id));
    return user || undefined;
  }

  async getUserByUsername(username: string): Promise<User | undefined> {
    const [user] = await db.select().from(users).where(eq(users.username, username));
    return user || undefined;
  }

  async createUser(insertUser: InsertUser): Promise<User> {
    const [user] = await db
      .insert(users)
      .values(insertUser)
      .returning();
    return user;
  }
  
  // Similar implementations for other methods: getStudySets, createFlashcard, etc.
}

//------------------------
// SERVER SETUP (server/index.ts)
//------------------------
import express, { type Request, Response, NextFunction } from "express";
import { registerRoutes } from "./routes";
import { setupVite, serveStatic, log } from "./vite";
import https from "https";
import http from "http";

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: false }));

// Logger middleware
app.use((req, res, next) => {
  // Logging middleware implementation
  next();
});

(async () => {
  const server = await registerRoutes(app);

  // Error handler
  app.use((err: any, _req: Request, res: Response, _next: NextFunction) => {
    const status = err.status || err.statusCode || 500;
    const message = err.message || "Internal Server Error";
    res.status(status).json({ message });
    throw err;
  });

  // Setup vite in development
  if (app.get("env") === "development") {
    await setupVite(app, server);
  } else {
    serveStatic(app);
  }

  // Start server
  const port = 5000;
  server.listen({
    port,
    host: "0.0.0.0",
    reusePort: true,
  }, () => {
    log(`serving on port ${port}`);
    
    // Setup a keep-alive mechanism
    const keepAliveInterval = 4 * 60 * 1000; // 4 minutes
    setInterval(() => {
      log("Keeping application alive...");
      // Ping mechanism implementation
    }, keepAliveInterval);
  });
})();

//------------------------
// REACT APP (client/src/App.tsx)
//------------------------
import { Switch, Route } from "wouter";
import { queryClient } from "./lib/queryClient";
import { QueryClientProvider } from "@tanstack/react-query";
import { Toaster } from "@/components/ui/toaster";
import NotFound from "@/pages/not-found";
import Home from "@/pages/home";
import Flashcards from "@/pages/flashcards";
import Quizzes from "@/pages/quizzes";
import Notes from "@/pages/notes";
import ProgressPage from "@/pages/progress";

function Router() {
  return (
    <Switch>
      {/* Home Page */}
      <Route path="/" component={Home} />
      
      {/* Flashcards Pages */}
      <Route path="/flashcards/:id" component={Flashcards} />
      
      {/* Quizzes Pages */}
      <Route path="/quizzes/:id" component={Quizzes} />
      
      {/* Notes Pages */}
      <Route path="/notes/:id" component={Notes} />
      
      {/* Progress Page */}
      <Route path="/progress" component={ProgressPage} />
      
      {/* Fallback to 404 */}
      <Route component={NotFound} />
    </Switch>
  );
}

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Router />
      <Toaster />
    </QueryClientProvider>
  );
}

export default App;# Study-life/
├── client/
│   └── src/
│       ├── components/
│       ├── hooks/
│       ├── lib/
│       ├── pages/
├── server/
├── shared/
