# Q&A System

Students have questions. Instructors have answers. A Q&A system bridges that gap, turning a one-way video course into an interactive learning experience. This chapter builds the question-and-answer feature where enrolled students can ask questions about courses, and instructors can respond.

---

## What you're building

The Q&A system consists of two main entities:

- **Question:** A student asks a question about a specific course or lesson
- **Answer:** An instructor (or the course creator) answers that question

The rules:
- Only enrolled students can ask questions
- Only the course instructor (or admin) can answer questions
- Questions belong to a course (optionally to a specific lesson)
- Answers belong to a question and an instructor
- Students can see all questions and answers for courses they're enrolled in
- Instructors see questions for their courses only

---

## The data model

Using MongoDB's native driver, you'll create two collections:

```javascript
// questions collection
{
  _id: ObjectId,
  course_id: ObjectId,        // which course this question is about
  lesson_id: ObjectId,        // optional: which specific lesson
  user_id: ObjectId,          // who asked (student)
  content: String,            // the question text
  created_at: Date,
  updated_at: Date
}

// answers collection
{
  _id: ObjectId,
  question_id: ObjectId,      // which question this answers
  instructor_id: ObjectId,    // who answered (instructor)
  content: String,            // the answer text
  created_at: Date,
  updated_at: Date
}
```

**Indexes to create:**
- `questions`: index on `course_id` (for listing questions by course)
- `questions`: index on `user_id` (for listing a student's questions)
- `questions`: compound index on `course_id, created_at` (for pagination)
- `answers`: index on `question_id` (for finding answers to a question)

---

## Authorization rules

Before writing code, define who can do what:

| Action | Who can do it | Why |
|---|---|---|
| Create question | Enrolled students only | Prevents spam from non-enrolled users |
| View questions | Enrolled students of that course | Privacy: students shouldn't see questions for courses they haven't paid for |
| Create answer | Course instructor or admin | Only the teacher should answer questions about their course |
| View answers | Enrolled students of that course | Same privacy as questions |
| Delete question | Question author or admin | Users can delete their own questions |
| Delete answer | Answer author or admin | Instructors can delete their own answers |

These rules prevent:
- Non-enrolled users from accessing course discussions
- Students answering other students' questions (instructor authority)
- Cross-course data leakage (IDOR prevention)

---

## API endpoints

Create these endpoints in `src/modules/qa/qa.routes.js`:

```
POST   /api/qa/questions           Create a question
GET    /api/qa/questions/:courseId List questions for a course (paginated)
GET    /api/qa/questions/:id       Get a single question with answers
POST   /api/qa/questions/:id/answers  Create an answer to a question
DELETE /api/qa/questions/:id       Delete a question
DELETE /api/qa/answers/:id         Delete an answer
```

---

## Implementation steps

**Step 1 — Create the Q&A module structure**

```bash
mkdir -p src/modules/qa
touch src/modules/qa/qa.routes.js
touch src/modules/qa/qa.service.js
touch src/modules/qa/qa.validators.js
```

**Step 2 — Write validators**

Use Zod to validate request bodies:

```js
// src/modules/qa/qa.validators.js
import { z } from 'zod';

export const createQuestionSchema = z.object({
  course_id: z.string().regex(/^[0-9a-fA-F]{24}$/, 'Invalid course ID'),
  lesson_id: z.string().regex(/^[0-9a-fA-F]{24}$/, 'Invalid lesson ID').optional(),
  content: z.string().min(10, 'Question must be at least 10 characters').max(1000)
});

export const createAnswerSchema = z.object({
  content: z.string().min(10, 'Answer must be at least 10 characters').max(2000)
});
```

**Step 3 — Write the service layer**

The service handles database operations using the MongoDB native driver:

```js
// src/modules/qa/qa.service.js
import { getDb } from '../lib/mongodb.js';
import { ObjectId } from 'mongodb';

export const qaService = {
  async createQuestion(userId, data) {
    const db = getDb();
    const question = {
      _id: new ObjectId(),
      course_id: new ObjectId(data.course_id),
      lesson_id: data.lesson_id ? new ObjectId(data.lesson_id) : null,
      user_id: new ObjectId(userId),
      content: data.content,
      created_at: new Date(),
      updated_at: new Date()
    };
    await db.collection('questions').insertOne(question);
    return question;
  },

  async getQuestionsByCourse(courseId, options = {}) {
    const db = getDb();
    const { limit = 20, skip = 0 } = options;
    
    const questions = await db.collection('questions')
      .find({ course_id: new ObjectId(courseId) })
      .sort({ created_at: -1 })
      .limit(limit)
      .skip(skip)
      .toArray();
    
    return questions;
  },

  async getQuestionWithAnswers(questionId) {
    const db = getDb();
    
    const question = await db.collection('questions')
      .findOne({ _id: new ObjectId(questionId) });
    
    if (!question) return null;
    
    const answers = await db.collection('answers')
      .find({ question_id: new ObjectId(questionId) })
      .sort({ created_at: 1 })
      .toArray();
    
    return { ...question, answers };
  },

  async createAnswer(questionId, instructorId, content) {
    const db = getDb();
    
    const answer = {
      _id: new ObjectId(),
      question_id: new ObjectId(questionId),
      instructor_id: new ObjectId(instructorId),
      content,
      created_at: new Date(),
      updated_at: new Date()
    };
    
    await db.collection('answers').insertOne(answer);
    return answer;
  },

  async deleteQuestion(questionId, userId) {
    const db = getDb();
    const result = await db.collection('questions').deleteOne({
      _id: new ObjectId(questionId),
      user_id: new ObjectId(userId)
    });
    return result.deletedCount > 0;
  },

  async deleteAnswer(answerId, instructorId) {
    const db = getDb();
    const result = await db.collection('answers').deleteOne({
      _id: new ObjectId(answerId),
      instructor_id: new ObjectId(instructorId)
    });
    return result.deletedCount > 0;
  }
};
```

**Step 4 — Write the routes**

Apply authorization middleware to enforce the rules:

```js
// src/modules/qa/qa.routes.js
import express from 'express';
import { authenticate } from '../../middleware/authenticate.js';
import { authorize } from '../../middleware/authorize.js';
import { qaService } from './qa.service.js';
import { createQuestionSchema, createAnswerSchema } from './qa.validators.js';

const router = express.Router();

// Create a question (enrolled students only)
router.post('/questions', authenticate, authorize('student'), async (req, res, next) => {
  try {
    const data = createQuestionSchema.parse(req.body);
    
    // Verify enrollment (check if user is enrolled in this course)
    const db = getDb();
    const enrollment = await db.collection('enrollments').findOne({
      student_id: new ObjectId(req.user.id),
      course_id: new ObjectId(data.course_id)
    });
    
    if (!enrollment) {
      return res.status(403).json({ error: 'You must be enrolled to ask questions' });
    }
    
    const question = await qaService.createQuestion(req.user.id, data);
    res.status(201).json(question);
  } catch (error) {
    next(error);
  }
});

// List questions for a course (enrolled students only)
router.get('/questions/:courseId', authenticate, async (req, res, next) => {
  try {
    const { courseId } = req.params;
    const { limit = 20, skip = 0 } = req.query;
    
    // Verify enrollment
    const db = getDb();
    const enrollment = await db.collection('enrollments').findOne({
      student_id: new ObjectId(req.user.id),
      course_id: new ObjectId(courseId)
    });
    
    if (!enrollment && req.user.role !== 'instructor' && req.user.role !== 'admin') {
      return res.status(403).json({ error: 'You must be enrolled to view questions' });
    }
    
    const questions = await qaService.getQuestionsByCourse(courseId, { limit: parseInt(limit), skip: parseInt(skip) });
    res.json(questions);
  } catch (error) {
    next(error);
  }
});

// Get a single question with answers
router.get('/questions/:id', authenticate, async (req, res, next) => {
  try {
    const question = await qaService.getQuestionWithAnswers(req.params.id);
    
    if (!question) {
      return res.status(404).json({ error: 'Question not found' });
    }
    
    // Verify enrollment or instructor/admin access
    const db = getDb();
    const enrollment = await db.collection('enrollments').findOne({
      student_id: new ObjectId(req.user.id),
      course_id: question.course_id
    });
    
    const isInstructor = req.user.role === 'instructor' && req.user.id.toString() === question.course_id.toString();
    const isAdmin = req.user.role === 'admin';
    
    if (!enrollment && !isInstructor && !isAdmin) {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    res.json(question);
  } catch (error) {
    next(error);
  }
});

// Create an answer (course instructor or admin only)
router.post('/questions/:id/answers', authenticate, authorize('instructor', 'admin'), async (req, res, next) => {
  try {
    const data = createAnswerSchema.parse(req.body);
    
    // Verify the question exists and belongs to a course this instructor teaches
    const db = getDb();
    const question = await db.collection('questions').findOne({
      _id: new ObjectId(req.params.id)
    });
    
    if (!question) {
      return res.status(404).json({ error: 'Question not found' });
    }
    
    // If instructor, verify they teach this course
    if (req.user.role === 'instructor') {
      const course = await db.collection('courses').findOne({
        _id: question.course_id,
        instructor_id: new ObjectId(req.user.id)
      });
      
      if (!course) {
        return res.status(403).json({ error: 'You can only answer questions for your courses' });
      }
    }
    
    const answer = await qaService.createAnswer(req.params.id, req.user.id, data.content);
    res.status(201).json(answer);
  } catch (error) {
    next(error);
  }
});

// Delete a question (author or admin only)
router.delete('/questions/:id', authenticate, async (req, res, next) => {
  try {
    if (req.user.role !== 'admin') {
      const deleted = await qaService.deleteQuestion(req.params.id, req.user.id);
      if (!deleted) {
        return res.status(403).json({ error: 'You can only delete your own questions' });
      }
    } else {
      // Admin can delete any question
      const db = getDb();
      await db.collection('questions').deleteOne({ _id: new ObjectId(req.params.id) });
    }
    
    res.status(204).send();
  } catch (error) {
    next(error);
  }
});

// Delete an answer (author or admin only)
router.delete('/answers/:id', authenticate, async (req, res, next) => {
  try {
    if (req.user.role !== 'admin') {
      const deleted = await qaService.deleteAnswer(req.params.id, req.user.id);
      if (!deleted) {
        return res.status(403).json({ error: 'You can only delete your own answers' });
      }
    } else {
      // Admin can delete any answer
      const db = getDb();
      await db.collection('answers').deleteOne({ _id: new ObjectId(req.params.id) });
    }
    
    res.status(204).send();
  } catch (error) {
    next(error);
  }
});

export default router;
```

**Step 5 — Register the routes**

In `src/app.js`:

```js
import qaRoutes from './modules/qa/qa.routes.js';

app.use('/api/qa', qaRoutes);
```

**Step 6 — Create indexes**

Create a script or add to your server startup to create the necessary indexes:

```js
// In src/server.js after connecting to MongoDB
async function createIndexes() {
  const db = getDb();
  
  await db.collection('questions').createIndex({ course_id: 1 });
  await db.collection('questions').createIndex({ user_id: 1 });
  await db.collection('questions').createIndex({ course_id: 1, created_at: -1 });
  await db.collection('answers').createIndex({ question_id: 1 });
  
  console.log('Database indexes created');
}

await createIndexes();
```

---

## Testing the endpoints

**Create a question:**
```bash
curl -X POST http://localhost:3000/api/qa/questions \
  -H "Authorization: Bearer <student_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "course_id": "507f1f77bcf86cd799439011",
    "content": "I don't understand the concept covered in lesson 3. Can you explain it differently?"
  }'
```

**List questions for a course:**
```bash
curl http://localhost:3000/api/qa/questions/507f1f77bcf86cd799439011 \
  -H "Authorization: Bearer <student_token>"
```

**Create an answer:**
```bash
curl -X POST http://localhost:3000/api/qa/questions/507f1f77bcf86cd799439012/answers \
  -H "Authorization: Bearer <instructor_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Great question! The concept works like this..."
  }'
```

---

## Definition of Done

- [ ] Students can create questions for courses they're enrolled in
- [ ] Students cannot create questions for courses they're not enrolled in
- [ ] Instructors can answer questions for their courses
- [ ] Instructors cannot answer questions for other instructors' courses
- [ ] Students can view questions and answers for enrolled courses only
- [ ] Question authors can delete their own questions
- [ ] Admins can delete any question or answer
- [ ] Pagination works for listing questions
- [ ] Database indexes are created for performance
- [ ] All endpoints have proper error handling
- [ ] Request validation prevents invalid data

All boxes ticked? Then continue to the next chapter — Search. If not, that's where today's work is.
