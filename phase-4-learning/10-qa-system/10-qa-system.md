# Q&A System

Students ask questions; instructors answer them. This is one of the highest-value features of a live course — it replaces the raised hand in a physical classroom.

---

## The models

Create `src/models/question.model.js` and `src/models/answer.model.js`:

```js
// question.model.js — shape only
const questionSchema = new mongoose.Schema(
  {
    courseId: { type: mongoose.Schema.Types.ObjectId, ref: 'Course', required: true },
    userId:   { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    content:  { type: String, required: true, trim: true, maxlength: 2000 },
  },
  { timestamps: true }
);

questionSchema.index({ courseId: 1, createdAt: -1 });
export const Question = mongoose.model('Question', questionSchema);
```

```js
// answer.model.js — shape only
const answerSchema = new mongoose.Schema(
  {
    questionId:   { type: mongoose.Schema.Types.ObjectId, ref: 'Question', required: true },
    instructorId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
    content:      { type: String, required: true, trim: true, maxlength: 5000 },
  },
  { timestamps: true }
);

export const Answer = mongoose.model('Answer', answerSchema);
```

---

## The endpoints

Scaffold `src/modules/qa/`.

| Method | Path | Access | What it does |
|--------|------|--------|-------------|
| `GET` | `/api/courses/:courseId/questions` | Enrolled student, Instructor (course owner) | List questions for a course, newest first |
| `POST` | `/api/courses/:courseId/questions` | Enrolled student | Ask a question |
| `POST` | `/api/questions/:questionId/answers` | Instructor (course owner) | Answer a question |

Key rules:
- Only enrolled students can ask questions. Check enrollment before allowing the post.
- Only the instructor who owns the course can answer. Check ownership.
- When returning questions, populate `userId` (name only) and include the answer if one exists.

---

## Definition of Done

- [ ] An enrolled student can post a question and see it in the list
- [ ] An unenrolled student attempting to post a question receives 403
- [ ] The course instructor can answer a question
- [ ] A different instructor attempting to answer receives 403 (not their course)
- [ ] `GET /api/courses/:courseId/questions` returns questions with author name and any existing answer
