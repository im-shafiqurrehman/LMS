# Admin Dashboard

The admin is the platform operator. They have a bird's-eye view of EduFlow: approving courses before they go live, monitoring platform health, and stepping in when things go wrong.

---

## The admin-only middleware

All admin routes require both `authenticate` (valid token) and `authorize('admin')` (role check):

```js
// In all admin routes
router.use(authenticate, authorize('admin'));
```

Apply this at the router level so every route in the admin module is protected automatically.

---

## Course approval workflow

When an instructor submits a course for review, its status changes from `draft` to `pending`. An admin reviews pending courses and either approves or rejects them.

`GET /api/admin/courses?status=pending`  
Returns all courses with `status: 'pending'`, populated with instructor details.

`POST /api/admin/courses/:id/approve`  
The service must:
1. Find the course: `Course.findById(id)`.
2. Confirm status is `pending` — reject other statuses with 422.
3. Set `status = 'published'` and call `course.save()`.
4. Clear the catalogue cache (invalidate `catalogue:*` Redis keys).
5. Send a notification email to the instructor (fire-and-forget).

`POST /api/admin/courses/:id/reject`  
Body: `{ reason }` (required)  
Set `status = 'rejected'`. Send rejection email with the reason.

---

## Platform metrics

`GET /api/admin/metrics`

Returns:

```json
{
  "totalUsers":       1240,
  "totalInstructors": 42,
  "totalCourses":     187,
  "publishedCourses": 143,
  "totalEnrollments": 8920,
  "revenueTotal":     445000
}
```

Use MongoDB aggregation pipelines for efficiency — one aggregate per collection, not six separate `.countDocuments()` calls:

```js
// Shape only
const [userStats] = await User.aggregate([
  {
    $group: {
      _id: null,
      total: { $sum: 1 },
      instructors: { $sum: { $cond: [{ $eq: ['$role', 'instructor'] }, 1, 0] } },
    },
  },
]);

const [enrollmentStats] = await Enrollment.aggregate([
  {
    $group: {
      _id: null,
      total: { $sum: 1 },
      revenue: { $sum: '$amountPaid' },
    },
  },
]);
```

---

## Definition of Done

- [ ] A student or instructor token attempting any `/api/admin/` route receives 403
- [ ] `GET /api/admin/courses?status=pending` returns pending courses with instructor details
- [ ] Approving a course sets status to `published` and clears the catalogue cache
- [ ] Rejecting a course sets status to `rejected`
- [ ] `GET /api/admin/metrics` returns accurate counts and revenue total

Write in `learning-log/14-admin-dashboard.md`:

1. Why must cache invalidation happen synchronously on approval rather than waiting for TTL expiry?
2. Why use an aggregation pipeline for the metrics query instead of multiple `countDocuments()` calls?
