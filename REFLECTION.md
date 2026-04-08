## Q1 — Middleware

In Express, middleware is a function that runs between receiving a request and sending a response. Middleware can inspect or change `req` and `res`, end the request early, or call `next()` to pass control to the next step.

The order of middleware matters because Express runs it from top to bottom. In my `server.js`, I register `cors()` and `express.json()` before the routes so that cross-origin requests are allowed and JSON request bodies are parsed before any route handler tries to use them. If `express.json()` were placed after a `POST` route, `req.body` might not be available when that route runs.

Global middleware is registered with `app.use()` and applies to every request in the application. In my server, these are global middleware:

```js
app.use(
  cors({
    origin: ALLOWED_ORIGINS,
  }),
);
app.use(express.json());
```

Route-level middleware only applies to a specific route. In my app, `verifyToken` is only used on the protected score submission route:

```js
app.post("/api/scores", verifyToken, async (req, res) => {
  // only logged-in users can submit scores
});
```

That middleware checks the JWT, adds the decoded user data to `req.user`, and then calls `next()`. This is why routes like `GET /api/scores` can stay public, while `POST /api/scores` is protected.

## Q2 — Password Security

Passwords must never be stored in plain text because if the database is breached, attackers would be able to see every user's actual password immediately. That creates a major security risk, especially because many users reuse the same password on other websites.

When `bcrypt.hash(password, 10)` is called, `bcrypt` creates a salted hash of the password rather than storing the password itself. The `10` is the `saltRounds` value, which controls how much processing work `bcrypt` uses when generating the hash. A higher number makes hashing slower, which helps protect against brute-force attacks.

`bcrypt.compare()` works by taking the password the user enters, hashing it again with the salt and settings stored in the original hash, and then checking whether the two hashes match. It does not reverse the hash, because hashes are designed to be one-way. Instead, it verifies the password by generating a new hash and comparing the result.

## Q3 — JWT Flow

1. **Registering an account:** The client sends an `email` and `password` to the registration route. The server checks that both fields are present, makes sure the password is at least six characters long, and checks whether that email is already in use. If everything is valid, the server hashes the password with `bcrypt` and stores the new user in MongoDB. It then returns a success response with a message, the user's ID, and their email.

2. **Logging in:** The client sends the `email` and `password` to the login route. The server looks up the user by email, compares the submitted password with the stored hashed password, and if they match, creates a JWT. The server then returns that token along with the user's ID and email. In my app, the JWT includes the user's `userId` and `email`, and it is signed with the server's secret key and an expiry time.

3. **Submitting a score:** After logging in, the client sends the score data to `POST /api/scores` and includes the JWT in the `Authorization` header as a Bearer token. The `verifyToken` middleware checks the token before the route runs. If the token is valid, the server reads the user information from the decoded token, saves the score to the database, and returns the newly created score record. If the token is missing or invalid, the server returns a `401 Unauthorized` response.

The server does not need to look up the database just to verify the token on each protected request because the JWT already contains the user's identifying information and is cryptographically signed. By using `jwt.verify()`, the server can confirm that the token is authentic and has not been tampered with, then trust the decoded payload in `req.user`.

## Q4 — In-Memory vs Database

The in-memory approach would cause two major problems. First, it does not persist data, so every time the server restarts or is redeployed, the scores array would be reset and all saved leaderboard data would disappear. Second, it does not enforce a proper schema, which makes the data less reliable because scores could be stored in inconsistent formats and would be harder to validate, sort, and manage as the app grows.

With MongoDB, the data is stored in the database instead of the server's temporary memory, so redeploying the server would not erase the saved scores. That is different from the in-memory array because the array only exists while the Node process is running, but MongoDB keeps the data separately and persists it across restarts.

## Q5 — Route Design Decision

This distinction makes sense because a leaderboard is meant to be easy for anyone to view, while submitting a score should only be allowed for authenticated users. Keeping `GET /api/scores` public allows all players to see rankings without needing to sign in, which improves usability and makes the app feel more open and accessible. Protecting `POST /api/scores` is important because it ensures the server knows who is submitting the score and helps prevent fake or unauthorized submissions.

If `GET /api/scores` also required authentication, it would create unnecessary friction for users who just want to view the leaderboard. New visitors would not be able to see scores unless they logged in first, which would make the app less convenient and reduce accessibility. It could also complicate the frontend by forcing it to manage tokens even for a simple read-only request.

Disclaimer: Responses were drafted then enhanced using GH Copilot.