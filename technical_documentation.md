# Technical documentation

## Lykiosbay services

Lykiosbay services is a REST API written in Java. It is a monolithic application containing all logic used in Lykiosbay.

### ADRs

| Technology | Description |
| --- | --- |
| Java 17 | Latest LTS java version at the moment with nice language features such as records and switch statement expressions |
| Spring boot 3.0 | The latest Spring boot version built upon java 17 bringing |
| Spring security | Default choice when setting up security in a Spring boot project |
| H2 Database | H2 is very commonly used as an in-memory relational database and works well with Spring** |
| Flyway database migration tool | Flyway aims for simplicity and convention over configuration which makes it very easy to use. Optionality to define callback scripts automatically populating the database |
| Mockito | Most popular mocking framework for Java which lets you define mocks annotation based |

** Due to unforeseen circumstances, one of the two engineers will not work on Lykiosbay. Because we now have less capacity than we first thought, we’ve chosen to reduce the scope of Lykiosbay as much as possible during the MVP phase.

### Architecture

#### **Security**

Since our api will be fully stateless, we have to work with JWT tokens for security.

The security is configured using the OAuth2 resource server which is built upon Spring security. It validates the token and blocks the resource if the token is not valid. This functionality could be written from scratch too by adding the security filters yourself but the OAuth2 resource server simplifies our job for a great deal.

Furthermore, we’ve implemented the `UserDetailsService` to authenticate users against our database. The loadUserByUsername searches the database for a user with that username and then uses composition to wrap this class as a `UserDetails`.

#### **Validation**

For validation, we use Spring boot validation. This enables you to use predefined validation annotations or create custom ones. The validator for an annotation kicks in when a parameter is annotated with `@Valid`.

Most of our validation is done with custom annotations and validators because this library lacked one crucial feature: the ability to configure them. The provided annotations take constants as attributes which cannot be fetched from our properties file.

Our solution was using the `spring boot configuration processor` dependency providing us with an easy way to get properties from the configuration files in our code. Then we defined constraint annotations which take a spEL (Spring Expression Language) as an attribute.

#### **Manipulation of images**

To manipulate the images of an auction, the client must always provide the current order of all pictures. This is done by providing a list of `AuctionImage` which have an id, order and name. Then the manipulation process kicks in.

First, we have to match the auction images with the uploaded files. Matching means checking that each auction image that isn’t stored yet, also has an uploaded file with the same name. In this matching process, a new list of auction images and a list of `StorableFile` is created. This is done to change the name of the image to a randomly generated unique name.

Now the new auction images get stored in the database, and the auction images that were not included in this list are removed. Also the files of the auction images that were not included anymore are deleted.

In the final step, the storable files that were created in the matching step are now saved.

The reason this is such a complicated process is that we need to change the filename of both the auction image and the uploaded file. After that we have to persist in the database first and finally save the images. The images cannot be saved before persisting in the database because saving images cannot be rolled back in the transaction.

#### **Making a bid**

Making bids is a critical point in our application so this process needs to be very smooth. Let's start with stating some prerequisites.

- The user can only bid on auctions that are ongoing
- The user cannot overbid himself/herself
- The user must have a higher bid than or equal to the minimum next bid
    - Opening price of the auction if no bids were made yet
    - Current highest bid + the minimum raise of the auction

So let’s say all these prerequisites are met, then we are done no?. Well, not exactly. What would happen if two users made a bid at exactly the same moment? Both validations could be met before the changes were written to the database, even if the second bid is lower than the first bid that will be written to the database. This is different than the expected behaviour of course.

The solution we’ve found is making use of an optimistic lock on the auctions table. An optimistic lock is actually just a counter on how many writes happened to this record. This lock can now be used as a validation when updating an entity in the database.

When the record is first selected, the optimistic lock column is also selected. Some operations are done and the entity is persisted again. In this update statement, the first value of the optimistic lock is included in the where clause. Would the record be modified in the meantime, the where on the optimistic lock would fail and throw an exception. This already solves the problem with bids overriding each other in the database. But what about the user experience?

If we wouldn’t change anything, one of the two bids would just fail. We want as many bids to go through, so we should retry making a bid again. `spring retry` is a very easy-to-use library providing the `@Retryable` aspect annotation. We configure this annotation to retry up to 4 times when a specific exception occurs, namely the `ObjectOptimisticLockingFailureException`.

So now, when whilst retrying, the second bid still meets the prerequisites and is higher than the updated minimum next bid, both bids are made in one go.

### Testing

For all tests, we try to eliminate boilerplate code and instantiating objects in the test itself. For objects that can be reused in tests, fixtures are created. These fixtures contain public static final properties which can be used throughout all tests. Note that if these objects are modified during a test, all tests using this object will be influenced by this. It's better to create a public static method in the fixture which can be called upon to create a new default instance.

#### **Unit (suffix Test)**

Unit testing are tests that do not start a Spring context. All dependencies are mocked or stubbed and only single units are tested. Evidently, MockMvc cannot be used in unit tests because this requires a Spring context to be started.

In the unit tests, Mockito is used to mock all dependencies with the `@Mock` annotation. It’s important that you verify all mocks to be called or not to be called after executing the method.

#### **Integration (suffix IT)**

In our project scope, integration tests are all tests that go further than unit tests and are required to start a Spring context. Some integration tests will only startup a slice of the application and others will initialise the whole Spring context. `@WebMvcTest` starts only the controller layer and more specifically the controllers passed to the annotation. `@DataJpaTest` will create a Spring context consisting of the repositories passed to the annotation.

Our integration tests can be splitted up into 4 types defined by the packages the test resides in:

- web
    
    `@WebMvcTest` for testing controllers and the mappers used in these controllers. Uses MockMvc to mock network requests.

- data

    `@DataJpaTest` for testing repositories and their real database interaction with our h2 test database. Changes made to the database during this test are always rolled back due to the `@DataJpaTest` annotation

- security

    To test the authorization of the application with the real security context. These are the only tests that use the real security context. Making request with MockMvc and a real JWT token that can be generated using the `TokenService`

- application

    End to end tests for the whole backend where the full Spring context is started. It mocks a request with MockMvc and then the database is checked to contain the correct data. These database changes are not automatically rolled back.

The team has created some meta annotations to declare the behaviour or configuration you want on that specific test:

- `@WebMvcTestNoSecurity`

    Extension of the `@WebMvcTest` that creates a context with all controllers, or the controllers passed to the annotation but disables the configured security on it so you don’t need to provide a JWT to hit endpoints.

    Activates the `test` profile.

- `@SpringBootTestNoSecurity`

    Extension of the `@SpringBootTest` that creates a context like the application would normally do except that the security will be turned off.
    
    Activates the `test` profile.

- `@JpaDatabaseTest`

    Extension of the `@DataJpaTest` but configures the test database for you.
    
    Activates the `test` profile.

- `@SecurityTest`

    The test will load the whole Spring context like it would do on a normal application startup. Only users will not be authenticated against the database but against our custom implementation of the `UserDetailsService` with static users.
    
    MockMvc will also be configured to use in these kinds of tests.
    
    Activates the `test-security` profile.

- `@EnableProperties`

    Loads the configuration properties in the context (use when creating slice tests because they don’t load properties)

- `@CleanDatabaseAfterTest`

    Tests configured with this annotation will run a clear tables script and then repopulate the database with the initial test data. This annotation should be used when doing a test modifying the database that will not be rolled back by default.

## Lykiosbay UI

Lykiosbay ui is a standalone React application made with Typescript. It uses the Lykiosbay services api to visualise the auctions and make bids.

### ADRs

| Technology | Description |
| --- | --- |
| React 18 | Latest React version at time of development, alternatives are Vue or Angular but the team is most experienced with React |
| React redux | Most widely used state management library for react and easy to use |
| Redux toolkit | Utilities to simplify common use cases inf redux and removing boilerplate code from plain redux code |
| RTK Query | Extension on redux toolkit allowing you to easily fetch and cache data automatically eliminating the need to hand-write data fetching and caching logic yourself |
| Moment.js | Simplifies parsing, validating, manipulating and displaying of dates in Javascript a great deal |

### Architecture

#### **Redux and backend communication**

The ui project is built to send the least amount possible of request to the backend. Technically, the application only sends one request to fetch the initial data, and from that point on, it can run without sending any more GET requests.

This is achieved by the power of Redux (toolkit) and RTK Query. At the startup of the application all data is fetched from the backend and the store is populated. From now on, only modifying requests will be made to the backend. These modifying requests will always include the updated entity in the response body as documented in the API spec of the backend. The store will then be updated with the new entity of the request body. Updating the store also triggers re-renders for React so ui updates are also handled automatically.

There are however some custom redux sliced apart from RTK Query. These slices are used to persist the token and user throughout multiple sessions via the localstorage.

#### **RTK Query abstraction**

RTK Query creates hooks to select states or send the requests to fetch the initial values of the state. These hooks are quite low level and will lead to a lot of boilerplate if they would be used directly in components.

That's why these hooks are wrapped in custom service hooks.

- `useUser`

    Selects the username from the store which was initialised from the localstorage or by logging in.
    
    Uses the useGetUserQuery hook to fetch the user data from the backend or cache if this hook was already executed.

    Returns the user.

- `useAuctions`

    Uses the previously defined useUser hook to get the current user.

    This user is used in the useGetAuctionsQuery from RTK Query to fetch all auctions from the backend or cache if this hook was already executed.

    Returns a list of all auctions.

- `useAuction`

    Uses the previously defined useAuctions hook to get all auctions. Expects an auction id as argument and then returns this auction from the auctions list.

#### **Validation**

For the frontend validation, no library fitted our needs to be very versatile, low-boilerplate and easy to implement. That’s why we’ve created a custom validation hook.

This hook is generic and returns an object of the type when the validation was successful. When the validation fails, it throws a `ValidationError` and the `errormessages` are updated per input field. You can pass a list of validators per field that have to act on it. Validators can also be created yourself by extending the `AbstractValidator`.

An example on how to use the custom validation hook. Let's say we want to create a form for logging in. The form should create an object of type `LoginRequest`.

```typescript
interface LoginRequest {
    username: string;
    password: string;
}
```

**1 - Define the input fields**
```html
<div>
  <label>Username</label>
  <input />
</div>
<div>
  <label>Password</label>
  <input type="password" />
</div>
```

**2 - Create references for each input field**

These references are used to read the current value of the input fields.

```typescript
const usernameRef = useRef<HTMLInputElement | null>(null);
const passwordRef = useRef<HTMLInputElement | null>(null);

...
```

```html
<div>
  <label>Username</label>
  <input
    ref={usernameRef}
  />
</div>
<div>
  <label>Password</label>
  <input
    type="password"
    ref={passwordRef}
  />
</div>
```

**3 - Use the useFormValidator hook and define the validators per field**

```typescript
const { getValidOrThrow, validationData } = useFormValidator<LoginRequest>({
    username: {
        fieldName: "Username",
        validators: [],
        reference: usernameRef
    },
    password: {
        fieldName: "Password",
        validators: [],
        reference: passwordRef
    }
});
```

The hook returns two functions:

- `getValidOrThrow`
    
    Returns the validated object of the type you passed to the hook.

- `validationData`
    React state containing fieldName, validators, reference and errorMessages.

    - Since this is a state, components rendering one of these values will automatically re-render if the value updates.

    - The error messages are an array of strings with each string being a validation error for that field.

The initial state of the hook must contain all fields of the type you passed to the hook, instead of a string, you have to give an object containing the following properties:

- `fieldName`
    
    Name of the field that should be used to create the error messages for that field.

- `validators`

    List of validator objects that should implement the Validator interface.

- `reference`
    
    React reference to the input field so that the validator can read the value of it.
 
**4 - Get the validated object**

You can now call the `getValidOrThrow` function that is returned by the hook. This will first execute the specified validators for each input field. If all fields are valid, the object created form the input fields is returned. When a validation for an input fails, a `ValidationError` is thrown.
 
**5 - Display the error messages per input**

You can just map the list of the `errorMessages` in the `validationData` returned by the hook.

These will trigger a re-render automatically when they change since they are a react state behind the scenes.

```html
<div>
  <label>Username</label>
  {validationData.username.errorMessages.map((errorMessage) => <div>{errorMessage}</div>)}
  <input
    ref={usernameRef}
  />
</div>
<div>
  <label>Password</label>
  {validationData.password.errorMessages.map((errorMessage) => <div>{errorMessage}</div>)}
  <input
    type="password"
    ref={passwordRef}
  />
</div>
```

### Testing

Testing the ui comes down to testing that all expected elements are rendered on the screen and all expected functions are called with their expected arguments.

#### **Unit**

Most tests will be unit tests because unit tests include testing components without the use of redux. In these unit tests, all dependencies and most important redux is mocked. 

#### **Integration (suffix IT)**

Integration tests will run using `msw` (mock service worker). With this library, we will mock the api responses to load the data in our redux store. These mocked endpoints can be found in the `__mocks__` package. We also test user interaction like clicking buttons or entering data in input fields.
