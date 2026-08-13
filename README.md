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

I chose to work on this artifact for this milestone and all of the milestones because I found the process of working through mobile applications entertaining. I discovered there was plenty of room for improvement in this application after reflecting on it with my increased knowledge of security and databases since that course. I would also like to be able to use this application to assist a hobby of mine, and I plan to eventually introduce more features that will better support my hobby. Mobile application development introduces multiple different software languages in a single environment. This inventory application uses at least three different languages: JavaScript, XML, and SQLite/C. Having multiple languages in a single application demonstrates my ability to switch between languages and their nuances, while showing that I can combine them to create a functional application. 

In this milestone, I focused on user authentication by changing the behavior of the user login function and adding a salted password hasher for protective password storage. Originally, the design of the login screen was built around a user logging into the application with their credentials. The application would check the credentials against the stored credentials in the database, and if they were a match, the screen would switch to the main menu. The only validation test was to check if the credentials matched, but now I have added checks for a blank username or password, password length requirements, and pre-existing usernames. 

The application now also stores login sessions, so the user stays logged in until the session is cleared or the user logs out, rather than starting at the login screen every time the application is launched. The application also now salts and hashes user passwords before storing them into the Room SQLite database. I chose to use OWASP’s recommended salted hash of PDKDF2-SHA256 with 600,000 iterations.

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

**Enhanced Code**

```java
public class RegisterActivity extends AppCompatActivity {
    private static final String TAG = "RegisterActivity";

    EditText usernameInput, passwordInput;
    Button registerButton;

    AuthenticationRepository authenticationRepository;
    // keeps password hashing and database work off the main thread
    private final ExecutorService authenticationExecutor =
            Executors.newSingleThreadExecutor();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_register);

        usernameInput = findViewById(R.id.usernameInput);
        passwordInput = findViewById(R.id.passwordInput);
        registerButton = findViewById(R.id.registerButton);

        authenticationRepository = AuthenticationProvider.create(this);

        // validates and creates the account in the background
        registerButton.setOnClickListener(view -> {
            String username = usernameInput.getText().toString();
            String password = passwordInput.getText().toString();

            // prevents duplicate registration requests
            registerButton.setEnabled(false);
            authenticationExecutor.execute(() -> {
                try {
                    AuthenticationRepository.Result result =
                            authenticationRepository.register(username, password);

                    // returns the registration result to the main thread
                    runOnUiThread(() -> showRegistrationResult(result));
                } catch (RuntimeException exception) {
                    Log.e(TAG, "Registration failed", exception);
                    runOnUiThread(this::showRegistrationError);
                }
            });
        });

    }
```

```java
public final class AuthenticationProvider {
    private AuthenticationProvider() {
    }

    public static AuthenticationRepository create(Context context) {
        return new AuthenticationRepository(
                new RoomUserStore(
                        AppDatabase.getInstance(context).userDao(),
                        new PasswordHasher()),
                new SharedPreferencesSessionStore(context)
        );
    }
}
```

```java
public final class PasswordHasher {

    private static final String ALGORITHM = "PBKDF2WithHmacSHA256";
    private static final String FORMAT_NAME = "pbkdf2-sha256";

    // configures the PBKDF2 work factor, salt size, and hash size
    private static final int ITERATIONS = 600_000;
    private static final int SALT_BYTES = 16;
    private static final int HASH_BYTES = 32;

    private final SecureRandom secureRandom = new SecureRandom();


    // creates a new salted hash for database storage
    public String hash(String password) {
        if (password == null || password.isEmpty()) {
            throw new IllegalArgumentException("Password cannot be null or empty");
        }

        byte[] salt = new byte[SALT_BYTES];
        // creates a unique random salt for every stored password
        secureRandom.nextBytes(salt);

        byte[] derivedHash = derive(password, salt, ITERATIONS);

        // stores the algorithm, iterations, salt, and hash in one value
        return FORMAT_NAME
                + "$" + ITERATIONS
                + "$" + encode(salt)
                + "$" + encode(derivedHash);
    }

    // checks an entered password against a stored salted hash
    public boolean verify(String password, String storedValue) {
        if (password == null || storedValue == null) {
            return false;
        }

        try {
            String[] fields = storedValue.split("\\$", -1);

            // rejects values that do not match the expected storage format
            if (fields.length != 4 || !FORMAT_NAME.equals(fields[0])) {
                return false;
            }

            int iterations = Integer.parseInt(fields[1]);

            // prevents malformed values from requesting excessive work
            if (iterations != ITERATIONS) {
                return false;
            }

            byte[] salt = decode(fields[2]);
            byte[] expectedHash = decode(fields[3]);

            if (salt.length != SALT_BYTES || expectedHash.length != HASH_BYTES) {
                return false;
            }

            byte[] suppliedHash = derive(password, salt, iterations);

            // compares hashes without revealing where they differ
            return MessageDigest.isEqual(expectedHash, suppliedHash);
        } catch (IllegalArgumentException exception) {
            // treats malformed numbers and Base64 as failed verification
            return false;

        }
    }

    private byte[] derive(String password, byte[] salt, int iterations) {
        PBEKeySpec specification = new PBEKeySpec(
                password.toCharArray(),
                salt,
                iterations,
                HASH_BYTES * 8
        );

        try {
            SecretKeyFactory factory = SecretKeyFactory.getInstance(ALGORITHM);
            return factory.generateSecret(specification).getEncoded();
        } catch (GeneralSecurityException exception) {
            throw new IllegalStateException(
                    "Cannot derive hash from password",
                    exception
            );
        } finally {
            // clears the password copy held by the key specification
            specification.clearPassword();
        }
    }

    private String encode(byte[] value) {
        return Base64.encodeToString(value, Base64.NO_WRAP);
    }

    private byte[] decode(String value) {
        return Base64.decode(value, Base64.NO_WRAP);
    }
}
```

**What Changed?**

The changes made in this section were to rebuild the registration activity in the application. This rebuild changed the method from raw username and password storage in the SQLite database local to the application to running the entered password through a password hashing algorithm that encrypted the password using PBKDF2-SHA256, as recommended by Android standards.

### Example 2: Login Activity using Hashed Passwords

**Original Code**

```java
public class LoginActivity extends AppCompatActivity {

    // --- UI Elements ---
    private EditText usernameInput;
    private EditText passwordInput;
    private Button submitButton;
    private Button createAccountButton;

    // --- Database ---
    private AppDatabase db;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.login_screen);

        // Initialize UI components
        usernameInput = findViewById(R.id.username);
        passwordInput = findViewById(R.id.password);
        submitButton = findViewById(R.id.submit_button);
        createAccountButton = findViewById(R.id.create_account_button);

        // Get database instance
        db = AppDatabase.getInstance(this);

        // Login button logic
        submitButton.setOnClickListener(view -> {
            String username = usernameInput.getText().toString();
            String password = passwordInput.getText().toString();

            // Check credentials against database
            User user = db.userDao().login(username, password);

            if (user != null) {
                Toast.makeText(this, "Login successful", Toast.LENGTH_SHORT).show();
                // Navigate to Main Menu on success
                startActivity(new Intent(this, MainMenuActivity.class));
            } else {
                Toast.makeText(this, "Invalid username or password", Toast.LENGTH_SHORT).show();
            }
        });

        // Navigate to Registration screen
        createAccountButton.setOnClickListener(view -> {
            startActivity(new Intent(this, RegisterActivity.class));
        });
    }
}
```

**Enhanced Code**

```java
public class LoginActivity extends AppCompatActivity {
    private static final String TAG = "LoginActivity";

    // stores the login screen controls
    private EditText usernameInput;
    private EditText passwordInput;
    private Button submitButton;
    private Button createAccountButton;

    private AuthenticationRepository authenticationRepository;
    // keeps password verification and database work off the main thread
    private final ExecutorService authenticationExecutor =
            Executors.newSingleThreadExecutor();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.login_screen);

        // connects the login screen controls
        usernameInput = findViewById(R.id.username);
        passwordInput = findViewById(R.id.password);
        submitButton = findViewById(R.id.submit_button);
        createAccountButton = findViewById(R.id.create_account_button);

        authenticationRepository = AuthenticationProvider.create(this);

        // skips login when a valid session already exists
        if (authenticationRepository.isSessionActive()) {
            openMainMenu();
            return;
        }

        // validates the credentials in the background when login is submitted
        submitButton.setOnClickListener(view -> {
            String username = usernameInput.getText().toString();
            String password = passwordInput.getText().toString();

            // prevents duplicate requests while authentication is running
            setAuthenticationControlsEnabled(false);
            authenticationExecutor.execute(() -> {
                try {
                    AuthenticationRepository.Result result =
                            authenticationRepository.login(username, password);

                    // returns the authentication result to the main thread
                    runOnUiThread(() -> showLoginResult(result));
                } catch (RuntimeException exception) {
                    Log.e(TAG, "Login failed", exception);
                    runOnUiThread(this::showLoginError);
                }
            });
        });

        // opens the registration screen
        createAccountButton.setOnClickListener(view -> {
            startActivity(new Intent(this, RegisterActivity.class));
        });
    }
```

**What Changed?**

The changes made in this section were to adjust the login activity from using the raw passwords entered by the user during login to encrypting the password and comparing the hashed password with the stored password from registration. I also changed the behavior of the application to use session handling, meaning after the user successfully logs into the application, a session is created. While the session is active, the application will reopen to the main menu after being closed and reopened. Before, the app would reopen to the login screen each time it was closed. The session will end when the user logs out.

### Reflection and Course Outcomes

Using my experience and newfound knowledge in security and OWASP standards, I learned a lot more about how this application’s login system could be improved and secured. I discovered how session handling works in Android applications and how to use Android's SecretKeyFactory to generate the salted hashed passwords using API version 26. I also got more acquainted with OWASP’s Password Storage Cheat Sheet to help me choose the parameters for the iterations, salt, and hash. I encountered challenges while learning how to implement these features and security standards because I needed to add an entirely new section to my application to handle session handling and password hashing.

After receiving feedback on my Milestone One assignment, I decided I needed to take a different route with this enhancement. I still improved the security of the application, but instead of working on the database itself, I changed the functionality of the user login, while adding proper input validation and duplicate account checks, as well as setting up a salted password hasher to protect the user account information while being transmitted and stored. This enhancement now covers course outcomes three, four, and five instead of just four and five as originally planned.

# Enhancement Two: Algorithms and Data Structures

## Enhancement Narrative

I chose this artifact for all three enhancement categories because I could see several areas where the original application could be improved to better showcase my skills and meet the course outcomes. For this enhancement, I focused on improving the organization and usability of the inventory list. Originally, the application pulled item data from the database into a List and displayed it using Android's RecyclerView. While this worked for a small inventory, the list was unsorted, unorganized, and did not include any search or filtering capabilities. As the inventory grew, finding and updating a specific item would become increasingly time-consuming.

To improve this, I added a category field to the Item table so users can assign categories to their inventory items. I also created a case-insensitive search bar that allows users to search by item name, description, or category. In addition, I added a category filter and sorting options that allow the inventory to be organized by name, category, or quantity in either ascending or descending order. The available categories are retrieved dynamically from the inventory and organized using a LinkedHashMap, which allows efficient key-based access while maintaining a predictable order.

Together, these changes make the inventory easier to navigate and more practical as the amount of stored data increases. This enhancement also demonstrates my ability to modify an existing database structure, work with data collections, implement searching, sorting, and filtering logic, and improve the overall usability and scalability of an application.

### Example 1: Category Field to Items

**Original Code**

```java
@Entity(tableName = "items")
public class Item {

    @PrimaryKey(autoGenerate = true)
    public int id;

    @NonNull
    public String name;
    public String description;
    public int quantity;

    public Item(@NonNull String name, String description, int quantity) {
        this.name = name;
        this.description = description;
        this.quantity = quantity;
    }

    public String getName() {
        return name;
    }

    public String getDescription() {
        return description;
    }

    public int getQuantity() {
        return quantity;
    }
}
```

**Enhanced Code**

```java
@Entity(tableName = "items")
public class Item {

    @PrimaryKey(autoGenerate = true)
    public int id;

    @NonNull
    public String name;
    public String description;
    public int quantity;
    @NonNull
    public String tag;

    public Item(@NonNull String name, String description, int quantity, @NonNull String tag) {
        this.name = name;
        this.description = description;
        this.quantity = quantity;
        this.tag = tag;
    }

    public String getName() {
        return name;
    }

    public String getDescription() {
        return description;
    }

    public int getQuantity() {
        return quantity;
    }

    public String getTag() {
        return tag;
    }
}
```
