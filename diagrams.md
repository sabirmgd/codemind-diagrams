# Codemind Architecture Diagrams

I'll create multiple diagrams to visualize the architecture and flow of the Codemind system based on the provided code.

## 1. High-Level System Architecture

This diagram shows the main components of the system and how they interact.

```mermaid
graph 
    Client[Client] -->|Makes API Request| Handler(handleOpenAIRequest)
    
    subgraph "Services"
        Handler -->|Uses| CodeReviewService[Code Review Service]
        CodeReviewService -->|Uses| OpenAiClient[OpenAI Client]
    end
    
    subgraph "VCS Clients"
        Handler -->|Uses| GitlabClient[GitLab Client]
        Handler -->|Uses| GithubClient[GitHub Client]
    end
    
    subgraph "Repositories"
        Handler -->|Uses| UserRepository[User Repository]
        Handler -->|Uses| SummaryRepository[Summary Repository] 
        Handler -->|Uses| FlagRepository[Flag Repository]
        CodeReviewService -->|Increments Credits| UserRepository
    end
    
    subgraph "External Services"
        OpenAiClient -->|Sends Requests| OpenAI[OpenAI API]
        GitlabClient -->|Interacts with| GitLabAPI[GitLab API]
        GithubClient -->|Interacts with| GitHubAPI[GitHub API]
    end
    
    subgraph "Database"
        Firebase[(Firebase Firestore)]
        UserRepository --> Firebase
        SummaryRepository --> Firebase
        FlagRepository --> Firebase
    end
```

## 2. Class Diagram - Main Components

This diagram shows the main classes and their relationships.

```mermaid
classDiagram
    class CodeReviewService {
        -openai
        -mode
        -userId
        +constructor(openai, mode, userId)
        +reviewCode(text)
        +reviewDiff(text)
        +summarizeDiff(text)
        +summarizeMr(summaries)
        +explainCode(text)
        +writeUnitTests(text)
        +findBugs(text)
        +optimizeCode(text, comment)
        +improveVariableNames(text)
        +splitIntoSmallerFunctions(text)
        +simplifyCode(text)
        +adHoc(text, comment)
        +summarizeCode(text)
    }

    class OpenAiClient {
        -openai
        +constructor(apiKey)
        +createChatCompletion(parameters, userId)
    }

    class GitlabClient {
        -api
        +constructor(apiKey)
        +getUserInformation(userId)
        +createCommentReply(gitProviderParams, comment)
        +createCommentInMR(gitProviderParams, comment)
        +createCommentInFile(gitProviderParams, filePath, comment, newLineLineNumber)
        +createCommentInFileForDiff(gitProviderParams, filePath, comment, oldLine, newLine)
    }

    class GitHubClient {
        -appId
        -privateKey
        -installationId
        -octokit
        +constructor(appId, privateKey, installationId)
        +initialize()
        +getUserInformation(username)
        +createCommentReply(gitProviderParams, comment)
        +createCommentInMR(gitProviderParams, comment)
        +createCommentInFile(gitProviderParams, filePath, comment, newEndLine, newStartLine)
        +createCommentInFileForDiff(gitProviderParams, filePath, comment, startLine, endLine)
        +getLatestCommitSha(owner, repo, pullNumber)
    }

    class SummaryRepository {
        +saveSummary(summaryData)
        +getSummariesByReviewId(reviewId)
    }

    class CodeReviewHelper {
        +splitNumberedPoints(text)
        +splitCodeContent(content)
        +prefixLinesWithNumbers(codeContent)
        +parseSuggestionFromLine(suggestion)
        +countLeadingSpaces(fullText, lineNumber)
        +formatSuggestionOutput(suggestionData, versionControl)
        +parseSuggestionsForFile(output)
        +parseSuggestionsForDiff(output)
        +extractLineNumberFromDiff(diff)
        +extractStartingLineNumbers(diff)
        +formatDiffWithNewLineNumbers(diff)
        +mapDiffLines(diff)
        +findStartLineForEndLine(lineMap, endLine)
    }
    
    CodeReviewService --> OpenAiClient : uses
    CodeReviewService --> CodeReviewHelper : uses
    GitlabClient --> UserRepository : uses for tokens
    GitHubClient --> UserRepository : uses for tokens
```

## 3. Sequence Diagram - Code Review Process

This diagram shows the sequence of actions when reviewing code.

```mermaid
sequenceDiagram
    participant Client
    participant Handler as handleOpenAIRequest
    participant CodeReviewService
    participant OpenAiClient
    participant VCSClient as GitlabClient/GitHubClient
    participant UserRepo as UserRepository

    Client->>Handler: Request code review
    Handler->>Handler: Validate request
    Handler->>UserRepo: getUserDetails(identifier)
    UserRepo-->>Handler: User details
    Handler->>VCSClient: Create client instance
    VCSClient-->>Handler: VCS client
    Handler->>OpenAiClient: Create OpenAI client
    OpenAiClient-->>Handler: OpenAI client
    Handler->>CodeReviewService: Create service with client and mode
    
    Note over Handler,CodeReviewService: Process based on type (block, file, diff) and feature
    
    Handler->>CodeReviewService: Review code/diff
    CodeReviewService->>CodeReviewHelper: Format code for review
    CodeReviewHelper-->>CodeReviewService: Formatted code
    CodeReviewService->>OpenAiClient: createChatCompletion(parameters, userId)
    OpenAiClient->>UserRepo: incrementCredit(userId, -tokensUsed)
    OpenAiClient-->>CodeReviewService: Review result
    CodeReviewService-->>Handler: Processed review
    
    Handler->>CodeReviewHelper: Parse suggestions
    CodeReviewHelper-->>Handler: Parsed suggestions
    
    Handler->>VCSClient: Create comment with suggestions
    VCSClient-->>Handler: Comment creation result
    
    Handler-->>Client: Response with review
```

## 4. Flowchart - Request Handler Decision Process

This flowchart shows the decision process in the request handler.

```mermaid
flowchart TD
    A[Request Received] --> B{Determine Request Type}
    B -->|Block| C[Process Block Request]
    B -->|File| D[Process File Request]
    B -->|Diff| E[Process Diff Request]
    
    C --> C1{Determine Feature}
    C1 -->|review/explain/etc| C2[Get review from CodeReviewService]
    C1 -->|optimize| C3[Format suggestion with line numbers]
    C2 --> C4[Create comment reply via VCS client]
    C3 --> C4
    
    D --> D1{Check Feature}
    D1 -->|review| D2[Get review from CodeReviewService]
    D1 -->|summarize| D3{Is Last Summary?}
    D2 --> D4[Handle code review for file]
    D3 -->|No| D5[Save summary to database]
    D3 -->|Yes| D6[Get stored summaries]
    D6 --> D7[Generate MR summary]
    D7 --> D8[Format final summary]
    D8 --> D9[Create comment in MR]
    
    E --> E1{Check Mode and Feature}
    E1 -->|MR + review| E2[Review single diff]
    E1 -->|MR + summarize| E3{Is Last Summary?}
    E3 -->|No| E4[Save summary to database]
    E3 -->|Yes| E5[Get stored summaries]
    E5 --> E6[Generate MR summary]
    E6 --> E7[Format final summary]
    E7 --> E8[Create comment in MR]
    
    E2 --> E9[Parse suggestions]
    E9 --> E10[Format suggestions]
    E10 --> E11[Create comments for diff]
```

## 5. Entity Relationship Diagram

This diagram shows the data entities and their relationships.

```mermaid
erDiagram
    USER {
        string id PK
        string gitlabId
        string installationId
        number credit
        string accessToken
        string refreshToken
    }
    
    SUMMARY {
        string reviewId PK
        string type
        string name
        string content
        string summary
        boolean isLast
        timestamp updatedAt
    }
    
    FLAG {
        string id PK
        boolean isEnabled
    }
    
    TOKEN {
        string id PK "apiRateLimit"
        number limitRequests
        number limitTokens
        number remainingRequests
        number remainingTokens
        string resetRequests
        string resetTokens
        timestamp updatedAt
    }
    
    USER ||--o{ SUMMARY : "creates"
    SUMMARY }|--|| FLAG : "may use"
```

## Summary of the Diagrams

1. **High-Level System Architecture**: Shows the main components of the system, including the handler, services, clients, repositories, and external services.

2. **Class Diagram**: Details the main classes in the system, their methods, and relationships.

3. **Sequence Diagram**: Illustrates the process flow for code review, from request to response.

4. **Flowchart**: Shows the decision-making process in the request handler.

5. **Entity Relationship Diagram**: Displays the data entities and their relationships in the Firebase Firestore database.

The system is a cloud-based service that processes code review requests, leveraging OpenAI's API to generate reviews and suggestions, and then sends those suggestions back through version control systems like GitLab and GitHub. The architecture follows a clean separation of concerns with dedicated components for each responsibility.
