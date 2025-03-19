# Cop-on Programming Manual

## Table of Contents
1. [Introduction to Context-Oriented Programming](#introduction)
2. [Language Fundamentals](#fundamentals)
3. [Context Definition and Management](#context-management)
4. [Functions and Context Handling](#functions)
5. [Type System and Context Inference](#type-system)
6. [Concurrency Model](#concurrency)
7. [Error Handling](#error-handling)
8. [Testing and Verification](#testing)
9. [AI Integration](#ai-integration)
10. [Standard Library](#standard-library)
11. [Best Practices](#best-practices)

<a name="introduction"></a>
## 1. Introduction to Context-Oriented Programming

Cop-on is a programming language designed around the principle of explicit context propagation. In Cop-on, context is a first-class citizen that represents the environment in which code executes. This approach combines functional purity with practical state management, creating a paradigm that is both theoretically sound and pragmatically useful.

### Core Principles

- **Explicit Context Propagation**: All functions receive a context as their first parameter
- **Contained State**: All mutable state resides within the context
- **Typed Dependencies**: The context provides typed access to dependencies
- **Controlled Side Effects**: All side effects are modeled as transformations of the context
- **Safe Concurrency**: Multiple execution flows can safely operate on checkpointed contexts

### When to Use Cop-on

Cop-on excels in scenarios requiring:
- Predictable state management
- AI workflow integration
- Concurrent processing with shared resources
- Systems where traceability and auditing are important
- Applications requiring checkpointing or migration of computation state

<a name="fundamentals"></a>
## 2. Language Fundamentals

### Basic Syntax

```
// Comment syntax uses double slashes
/* Or block comments */

// Module declaration
module MyModule {
    // Code goes here
}

// Context type declaration
context AppContext {
    db: Database;
    logger: Logger;
    config: Config;
}

// Function with context parameter
fn process(ctx: AppContext, data: String) -> Result {
    // Function body
}
```

### Hello World Example

```
context MinimalContext {
    console: Console;
}

fn main(ctx: MinimalContext) -> () {
    ctx.console.println("Hello, Context-Oriented World!");
}
```

<a name="context-management"></a>
## 3. Context Definition and Management

### Defining Contexts

Contexts are defined using the `context` keyword:

```
context DatabaseContext {
    db: Database;
    connectionPool: ConnectionPool;
    queryCache: Cache<String, Result>;
}
```

### Combining Contexts

Contexts can be combined using the `with` operator:

```
context LoggingContext {
    logger: Logger;
    metrics: MetricsCollector;
}

context AppContext = DatabaseContext with LoggingContext;
```

### Context Access

Context members are accessed directly through the context variable:

```
fn queryUser(ctx: DatabaseContext, userId: String) -> User? {
    return ctx.db.query("SELECT * FROM users WHERE id = ?", userId);
}
```

### Type Inference

Cop-on features strong type inference for contexts:

```
// Type-inferred context parameter
fn log(ctx, message: String) -> () {
    // The compiler infers that ctx must contain a logger
    ctx.logger.log(message);
}
```

<a name="functions"></a>
## 4. Functions and Context Handling

### Function Declaration

Functions in Cop-on always take a context as their first parameter:

```
fn calculateTotal(ctx: PricingContext, items: List<Item>) -> Money {
    let total = Money(0, ctx.currencyService.defaultCurrency());
    
    for item in items {
        let price = ctx.pricingEngine.getPrice(item);
        total = total.add(price);
    }
    
    return total;
}
```

### Context Transformation

Functions that modify the context must indicate this in their signature:

```
fn cacheResult(ctx ->> ctx': CacheContext, key: String, value: Any) -> () {
    // The ->> operator indicates context transformation
    ctx'.cache.put(key, value);
}
```

### Function Overloading by Context

Functions can be overloaded based on their context type:

```
fn process(ctx: ValidationContext, data: Data) -> ValidationResult {
    // Validation processing
}

fn process(ctx: ProductionContext, data: Data) -> ProcessingResult {
    // Production processing
}
```

### Safe Context Updates

Context should only be updated at safe points:

```
fn transferMoney(ctx ->> ctx': BankingContext, from: Account, to: Account, amount: Money) -> TransferResult {
    // Validate before modifying context
    if !ctx.accountService.canWithdraw(from, amount) {
        return TransferResult.insufficientFunds();
    }
    
    // Update context atomically
    atomic {
        ctx'.accountService.withdraw(from, amount);
        ctx'.accountService.deposit(to, amount);
        ctx'.transactionLog.record(from, to, amount);
    }
    
    return TransferResult.success();
}
```

<a name="type-system"></a>
## 5. Type System and Context Inference

### Basic Types

Cop-on includes standard primitive types:
- `Int`, `Float`, `Double`, `String`, `Boolean`
- Collection types: `List<T>`, `Set<T>`, `Map<K, V>`
- Optional type: `T?`
- Result type: `Result<T, E>`

### Context Type Rules

Each type can appear only once in a context:

```
// Valid
context ValidContext {
    logger: Logger;
    db: Database;
}

// Invalid - Database appears twice
context InvalidContext {
    primaryDb: Database;
    backupDb: Database;  // Error: Type Database already exists in context
}

// Solution - Use type wrappers
context FixedContext {
    primaryDb: Primary<Database>;
    backupDb: Backup<Database>;
}
```

### Type Inference for Contexts

The compiler automatically infers minimum required context types:

```
fn processUser(ctx, userId: String) -> UserProfile {
    let user = ctx.userRepository.findById(userId);  // Infers ctx needs UserRepository
    ctx.logger.info("Processing user: ${userId}");   // Infers ctx needs Logger
    return user.toProfile();
}

// Compiler infers that this function requires:
// context { userRepository: UserRepository; logger: Logger; }
```

<a name="concurrency"></a>
## 6. Concurrency Model

### Spawning Concurrent Tasks

Concurrent execution is managed through context spawning:

```
fn processItems(ctx: ProcessingContext, items: List<Item>) -> List<Result> {
    let results = List<Result>();
    
    for item in items {
        // Create a checkpoint of the context for concurrent processing
        let itemCtx = ctx.checkpoint();
        
        // Spawn a new task with the checkpointed context
        spawn processItem(itemCtx, item) -> result {
            results.add(result);
        }
    }
    
    // Wait for all processing to complete
    await all;
    
    return results;
}
```

### Context Synchronization

Contexts can be merged after concurrent processing:

```
fn validateAndProcess(ctx ->> ctx': AppContext, document: Document) -> ProcessResult {
    // Create checkpoints for concurrent validation and processing
    let validationCtx = ctx.checkpoint();
    let processingCtx = ctx.checkpoint();
    
    // Spawn concurrent tasks
    let validationResult = spawn validate(validationCtx, document);
    let processingResult = spawn preprocess(processingCtx, document);
    
    // Wait for both tasks to complete
    await all;
    
    if validationResult.isError() {
        return ProcessResult.validationError(validationResult.error());
    }
    
    // Merge the contexts back
    ctx' = ctx.merge(validationCtx, processingCtx);
    
    return processResult;
}
```

### Async Function Handling

In JavaScript or Python integration, Cop-on functions use async/await:

```
// JavaScript/TypeScript integration
async fn fetchUserData(ctx: ApiContext, userId: String) -> UserData {
    const response = await ctx.httpClient.get(`/users/${userId}`);
    ctx.metrics.recordApiCall("fetchUser");
    return UserData.fromJson(response.body);
}
```

<a name="error-handling"></a>
## 7. Error Handling

### Result Types

Errors are returned explicitly through result types:

```
fn divideNumbers(ctx: MathContext, a: Float, b: Float) -> Result<Float, DivisionError> {
    if b == 0 {
        return Result.error(DivisionError.zeroDivisor());
    }
    
    return Result.success(a / b);
}
```

### Using Results

Results are handled explicitly in the calling code:

```
fn calculateRatio(ctx: AppContext, x: Float, y: Float) -> Float {
    let result = divideNumbers(ctx, x, y);
    
    if result.isError() {
        ctx.logger.error("Division error: ${result.error()}");
        return 0.0;
    }
    
    return result.value();
}
```

### Error Propagation

Errors can be propagated using the `try` operator:

```
fn processRatio(ctx: AppContext, a: Float, b: Float) -> Result<Float, ProcessingError> {
    // The try operator unwraps a Result or propagates the error
    let ratio = try divideNumbers(ctx, a, b)
        .mapError(e -> ProcessingError.calculationError(e));
    
    return Result.success(ratio * 100);
}
```

<a name="testing"></a>
## 8. Testing and Verification

### Mock Contexts

Tests can use mock contexts to isolate functionality:

```
test "should calculate total price correctly" {
    // Create a mock context
    let ctx = MockContext {
        pricingEngine: MockPricingEngine {
            getPrice: fn(item: Item) -> Money {
                return Money(item.basePrice * 1.2, "USD");
            }
        },
        currencyService: MockCurrencyService {
            defaultCurrency: fn() -> String { return "USD"; }
        }
    };
    
    let items = [Item("A", 10.0), Item("B", 15.0)];
    let total = calculateTotal(ctx, items);
    
    assert.equals(total.amount, 30.0);
    assert.equals(total.currency, "USD");
}
```

### Context Assertions

Invariants can be defined on contexts:

```
context BankingContext {
    accounts: AccountRepository;
    transactions: TransactionLog;
    
    invariant nonNegativeBalances {
        for account in accounts.all() {
            assert account.balance >= 0;
        }
    }
}
```

### Pre and Post Conditions

Functions can specify pre and post conditions:

```
fn transferFunds(ctx ->> ctx': BankingContext, from: Account, to: Account, amount: Money) -> TransferResult 
    requires {
        amount > 0;
        from.balance >= amount;
    }
    ensures {
        from.balance == old(from.balance) - amount;
        to.balance == old(to.balance) + amount;
    }
{
    // Implementation
}
```

<a name="ai-integration"></a>
## 9. AI Integration

### AI Model Context

AI models can be included in context definitions:

```
context LlmContext {
    model: LanguageModel;
    promptTemplates: Map<String, String>;
    tokenizer: Tokenizer;
}

fn generateResponse(ctx: LlmContext, userQuery: String) -> String {
    let prompt = ctx.promptTemplates.get("customerSupport")
        .replace("{query}", userQuery);
    
    return ctx.model.complete(prompt, ctx.tokenizer);
}
```

### AI Collaboration

Multiple AI models can collaborate through context passing:

```
fn collaborativeGeneration(ctx: AiCollaborationContext, task: Task) -> Solution {
    // Create task-specific context
    let taskCtx = ctx.withTask(task);
    
    // First AI plans the approach
    let plan = ctx.plannerModel.createPlan(taskCtx);
    
    // Add plan to context
    let executionCtx = taskCtx.withPlan(plan);
    
    // Second AI executes the plan
    let solution = ctx.executorModel.implement(executionCtx);
    
    // Third AI reviews the solution
    let review = ctx.reviewerModel.evaluate(executionCtx.withSolution(solution));
    
    if review.requiresRevision {
        return collaborativeGeneration(
            ctx.withFeedback(review.feedback), 
            task
        );
    }
    
    return solution;
}
```

### Safety Checks

AI operations can include safety checks through context:

```
fn generateContent(ctx ->> ctx': ContentGenerationContext, prompt: String) -> Content {
    // Safety check before generation
    let safetyCheck = ctx.safetyFilter.evaluate(prompt);
    if !safetyCheck.safe {
        ctx'.metrics.recordSafetyViolation(prompt, safetyCheck.reason);
        return Content.rejected(safetyCheck.reason);
    }
    
    // Generate content
    let content = ctx.llmModel.generate(prompt);
    
    // Post-generation safety check
    let contentCheck = ctx.safetyFilter.evaluate(content.text);
    if !contentCheck.safe {
        ctx'.metrics.recordPostGenerationViolation(content.text, contentCheck.reason);
        return Content.rejected(contentCheck.reason);
    }
    
    ctx'.metrics.recordSuccessfulGeneration(prompt);
    return content;
}
```

<a name="standard-library"></a>
## 10. Standard Library

Cop-on includes a standard library of common contexts and utilities:

### IO Context

```
context IoContext {
    fileSystem: FileSystem;
    console: Console;
    network: NetworkClient;
}

fn readAndProcessFile(ctx: IoContext, path: String) -> ProcessingResult {
    let content = ctx.fileSystem.readFile(path);
    let processed = process(ctx, content);
    ctx.console.println("Processed ${path}: ${processed.summary}");
    return processed;
}
```

### Database Context

```
context DbContext {
    connection: DatabaseConnection;
    transactionManager: TransactionManager;
    queryBuilder: QueryBuilder;
}

fn saveEntity(ctx ->> ctx': DbContext, entity: Entity) -> Result<Id, DbError> {
    // Begin transaction
    let tx = ctx.transactionManager.begin();
    ctx' = ctx.withTransaction(tx);
    
    try {
        let query = ctx.queryBuilder.insert(entity);
        let result = ctx.connection.execute(query);
        tx.commit();
        return Result.success(result.generatedId);
    } catch e {
        tx.rollback();
        return Result.error(DbError.fromException(e));
    }
}
```

### HTTP Context

```
context HttpContext {
    client: HttpClient;
    rateLimit: RateLimiter;
    responseCache: Cache<String, Response>;
}

fn fetchWithCaching(ctx ->> ctx': HttpContext, url: String) -> Response {
    // Check cache first
    if ctx.responseCache.has(url) {
        return ctx.responseCache.get(url);
    }
    
    // Check rate limit
    ctx.rateLimit.waitIfNeeded();
    
    // Make request
    let response = ctx.client.get(url);
    
    // Update cache
    ctx'.responseCache.put(url, response);
    
    return response;
}
```

<a name="best-practices"></a>
## 11. Best Practices

### Context Design Principles

1. **Single Responsibility**: Each context should have a clear, focused purpose
2. **Minimal Scope**: Include only the dependencies needed for the operations
3. **Logical Cohesion**: Group related dependencies in specialized contexts
4. **Clear Boundaries**: Define clear context boundaries for different domains

### Function Design

1. **Pure When Possible**: Keep functions pure (no context mutation) when feasible
2. **Explicit Transformation**: Make context transformations explicit and obvious
3. **Safe Points**: Update contexts only at well-defined safe points
4. **Result Clarity**: Return clear result types that express success or failure

### Context Flow Visualization

The recommended way to visualize context flow is as a lattice diagram:

```
              Initial Context
                    │
          ┌─────────┴─────────┐
          │                   │
input1 → Func1 → result1   input2 → Func2 → result2
          │                   │
          └─────────┬─────────┘
                    │
              Updated Context
                    │
                  Func3 → result3
                    │
              Final Context
```

This represents how context flows from top to bottom while regular inputs/outputs flow from left to right.

### Safety Guidelines

1. **Invariant Protection**: Define and enforce context invariants
2. **Atomic Updates**: Use atomic blocks for related context changes
3. **Checkpointing**: Create context checkpoints before risky operations
4. **Graceful Degradation**: Design contexts to handle partial failures

### AI-Specific Best Practices

1. **Model Isolation**: Keep AI models in dedicated context components
2. **Safety Layering**: Apply both pre and post safety checks
3. **Feedback Loops**: Design context to capture and utilize feedback
4. **Traceability**: Log context transitions for AI decision auditing
