# Computer Science Capstone ePortfolio

Welcome to my professional ePortfolio. This portfolio demonstrates my growth in software design, engineering, algorithms, databases, security, and professional communication.

## Code Review

Before I started working on enhancing my chosen artifact, I performed an informal code review of the original artifact. This code review examines the existing code and analyzes it based on its existing design, functionality, and opportunities for improvement. Since I decided to work on one artifact for all three enhancements, I have evaluated different sections of the original artifact that are relevant to the planned enhancements.

[![Watch My Code Review](https://img.youtube.com/vi/JozCU0tl3H0/0.jpg)](https://www.youtube.com/watch?v=JozCU0tl3H0)

**[Watch the Code Review on YouTube](https://www.youtube.com/watch?v=JozCU0tl3H0)**

## Original Artifact

### Artifact Overview

**Artifact:** Inventory Management Application
**Originating Course:** CS360: Mobile Architect and Programming
**Date Created:** April 23rd, 2026

The artifact that I worked with in this milestone is an inventory management application that I developed while in my mobile application development and design course at SNHU. The final version of this application was completed in April of 2026. This inventory application had a fairly simple set of goals it was required to meet: user logins, a dynamic item inventory, and SMS notifications. By the end of my course, I had most of the application functional, and I was able to use it as a basic inventory application with minimal security and database functionality.

[View/Download the Original Artifact](https://github.com/MeestaMoose/meestamoose.github.io/blob/b643fc73297168faf21122d205b819c2fa4b24b7/Project%203%20HHewitt.zip)

# Enhancement One: Software Design and Engineering

## Enhancement Narrative

I chose to work on this artifact for this milestone and all of the milestones because I found the process of working through mobile applications entertaining. I discovered there was plenty of room for improvement in this application after reflecting on it with my increased knowledge of security and databases since that course. I would also like to be able to use this application to assist a hobby of mine, and I plan to eventually introduce more features that will better support my hobby. Mobile application development introduces multiple different software languages in a single environment. This inventory application uses at least three different languages: JavaScript, XML, and SQLite/C. Having multiple languages in a single application demonstrates my ability to switch between languages and their nuances, while showing that I can combine them to create a functional application. In this milestone, I focused on user authentication by changing the behavior of the user login function and adding a salted password hasher for protective password storage. Originally, the design of the login screen was built around a user logging into the application with their credentials. The application would check the credentials against the stored credentials in the database, and if they were a match, the screen would switch to the main menu. The only validation test was to check if the credentials matched, but now I have added checks for a blank username or password, password length requirements, and pre-existing usernames. The application now also stores login sessions, so the user stays logged in until the session is cleared or the user logs out, rather than starting at the login screen every time the application is launched. The application also now salts and hashes user passwords before storing them into the Room SQLite database. I chose to use OWASP’s recommended salted hash of PDKDF2-SHA256 with 600,000 iterations.

## Code Changes

### Example 1: Salted and Hashed Password Storage

**Original Code**

```java
public class RegisterActivity extends AppCompatActivity {

    EditText usernameInput, passwordInput;
    Button registerButton;

    AppDatabase db;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_register);

        usernameInput = findViewById(R.id.usernameInput);
        passwordInput = findViewById(R.id.passwordInput);
        registerButton = findViewById(R.id.registerButton);

        db = AppDatabase.getInstance(this);

        registerButton.setOnClickListener(view -> {
            String username = usernameInput.getText().toString();
            String password = passwordInput.getText().toString();

            User user = new User(username, password);
            db.userDao().insert(user);

```
