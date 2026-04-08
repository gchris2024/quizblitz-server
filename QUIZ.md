## Q1

B) app.use(express.json()) middleware is missing or registered after the route

## Q2

- **400 Bad Request** means the client sent invalid or incomplete data.  
  **Example:** In QuizBlitz, this would be correct if a user tries to register without an `email` or `password`, or submits a password that is too short.

- **401 Unauthorized** means the request requires valid authentication, but the client did not provide a valid token or credentials.  
  **Example:** In QuizBlitz, this would be correct if someone tries to `POST /api/scores` without a JWT or with an invalid or expired token.

- **404 Not Found** means the client requested a route or resource that does not exist on the server.  
  **Example:** In QuizBlitz, this would be correct if someone tries to access an endpoint like `/api/leaderboard` when that route is not defined.

## Q3

The problem is that the database query is asynchronous, but the code does not wait for it to complete before sending a response. As a result, `res.json()` is called immediately, and the query result is never used.

Corrected version using `async/await`:

```js
app.get("/api/scores", async (req, res) => {
  try {
    const scores = await Score.find().sort({ score: -1 }).limit(10);
    res.json(scores);
  } catch (error) {
    console.error("Error fetching scores:", error.message);
    res.status(500).json({ error: "Failed to fetch scores" });
  }
});
```

## Q4

B) A schema defines the shape and validation rules for documents; a model is the class you use to query and save documents based on that schema

## Q5

One genuine advantage of the cookie approach is convenience, because the browser can automatically send the cookie with each request after login. One genuine advantage of the Authorization header approach is that it gives the client more explicit control over authentication, which is especially useful for APIs used by mobile apps or different frontends.

For a mobile-accessible game like QuizBlitz, the Authorization header approach is more appropriate. It works well for both browser and mobile clients, and it fits a token-based API design where the frontend stores the JWT and sends it only when needed. It is also simpler to manage consistently across different platforms.

Disclaimer: Responses were drafted then enhanced using GH Copilot.
