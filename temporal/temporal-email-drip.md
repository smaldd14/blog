# Building an Email Drip Campaign with Temporal.io

Email drip campaigns are essential for customer onboarding, but building reliable, long-running email sequences can be challenging. Traditional approaches often struggle with failures, retries, and maintaining state across extended time periods.

I've been using [Temporal.io](https://temporal.io) extensively in my work, and it's perfect for this use case. Temporal allows you to "code to the happy path" while automatically handling retries, failures, and workflow durability.

In this article, we'll explore how to build a robust email drip campaign using:
- **Temporal.io** for workflow orchestration
- **Spring Boot** for the application framework  
- **Temporal Java SDK** for workflow implementation
- **AWS SES** for email delivery
- **PostgreSQL** for data persistence

The complete implementation is open source and available [here](https://github.com/smaldd14/onboard-flow).

## What is Temporal

There are many great articles out there from people much smarter than I, like former Temporal employee [Swyx](https://x.com/swyx), discussing Temporal [on Hacker News](https://news.ycombinator.com/item?id=30365798), or his [bull case for Temporal](https://www.swyx.io/temporal-centicorn), or how Temporal used OpenAI's Codex CLI to [improve the Java SDK](https://temporal.io/blog/improving-java-sdk-codex-openai).

What makes Temporal powerful for email campaigns:

- **Workflow-as-code**: Your business logic is clearly expressed in code, not hidden in configuration
- **Automatic retries**: Failed email sends are automatically retried with exponential backoff
- **Durability**: Workflows survive server restarts and can run for weeks or months
- **Observability**: Built-in monitoring and debugging tools
- **Open source**: Self-hostable with no vendor lock-in 

## Key Temporal Concepts

Before diving into the implementation, let's understand Temporal's core concepts:

- **Workflows**: Define the overall business logic and orchestrate the sequence of steps
- **Activities**: Individual tasks that perform actual work (like sending emails or database operations)
- **Workers**: Processes that execute workflows and activities

**Activities** are designed to be idempotent—running them multiple times should produce the same result. This is crucial for reliability when Temporal automatically retries failed operations. In our email campaign, activities handle tasks like:
- Loading email sequences from the database
- Checking user subscription status  
- Sending individual emails
- Logging delivery events

## Implementation Overview

Here's a simplified version of our email drip workflow:
```java
@WorkflowImpl(taskQueues = TASK_QUEUE)
public class OnboardingWorkflowImpl implements OnboardingWorkflow {
    private static final String TASK_QUEUE = "onboarding-task-queue";

    private final EmailActivities activities = Workflow.newActivityStub(
        EmailActivities.class,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofMinutes(2))
            .setRetryOptions(
                io.temporal.common.RetryOptions.newBuilder()
                    .setMaximumAttempts(3)
                    .setInitialInterval(Duration.ofSeconds(1))
                    .setMaximumInterval(Duration.ofMinutes(1))
                    .build()
            )
            .build()
    );

    // Workflow starting point
    @Override
    public void executeOnboardingSequence(OnboardingWorkflowInput input) {
        activities.updateProgress(OnboardingStatus.STARTED);
        var sequenceConfig = activities.loadEmailSequence(input.sequenceId());

        if (sequenceConfig == null) {
            // email sequence not found, nothing to run
            activities.updateProgress(OnboardingStatus.CANCELLED);
            return;
        }

        for (var step : sequenceConfig.steps()) {
            if (!shouldContinue()) {
                break; // stop email sequence if user cancelled, workflow was paused, or user converted
            }

            if (step.delayFromStart() != 0) {
                // wait to send email
                Workflow.sleep(step.delayFromStart());
            }

            if (!shouldContinue()) {
                break; // stop email sequence if user cancelled, workflow was paused, or user converted
            }

            activities.checkEmailConditions(step);

            activities.sendEmail(step);

            activities.logEmailEvent(step.toEmailEventInput());

        }
    }
}
```

The actual implementation has more error handling and features, but you can see the core business logic clearly defined:

### Workflow Steps Explained

The workflow follows a clear sequence:

1. **Initialize**: Mark the campaign as started
2. **Load Configuration**: Retrieve the email sequence from the database
3. **Validate**: Ensure the sequence exists and is properly configured
4. **Process Each Email**: For every email in the sequence:
   - **Pre-check**: Verify user hasn't unsubscribed or converted
   - **Wait**: Sleep until the scheduled send time (e.g., "3 days after signup")
   - **Post-sleep check**: Re-verify user status after the delay
   - **Condition check**: Ensure user still meets sending criteria
   - **Send**: Deliver the email via AWS SES
   - **Log**: Record the delivery event for analytics

This approach ensures emails are sent at the right time to the right users, with automatic handling of failures and edge cases.

### Error Handling and Reliability

Temporal provides robust error handling out of the box:

- **Automatic retries**: Failed activities are retried with configurable backoff strategies
- **Failure isolation**: A single email failure doesn't stop the entire campaign
- **Workflow durability**: Campaigns survive server restarts and deployments
- **Observability**: Monitor workflow progress and debug issues through Temporal's UI

For complex scenarios, you can implement the [Saga pattern](https://docs.temporal.io/encyclopedia/application-design-patterns#saga) for cleanup operations, though it's not needed for basic email campaigns.

Learn more about [Temporal's failure handling](https://docs.temporal.io/concepts/what-is-a-failure).

## Application Architecture

The email drip campaign application is built with a modern, scalable architecture:

### Tech Stack
- **Spring Boot** - Application framework
- **PostgreSQL** - Database for templates and sequences
- **JPA/Hibernate** - Data persistence layer
- **Temporal Java SDK** - Workflow orchestration
- **AWS SES** - Email delivery service

### Key Features
- Customer onboarding workflow management
- Database-backed email template management with full CRUD operations
- Dynamic email sequence creation and management
- Template preview and validation system
- Automated email sequences with AWS SES
- Temporal.io workflow orchestration
- PostgreSQL database with JPA/Hibernate
- Comprehensive REST APIs for onboarding, templates, and sequences
- Email event tracking and analytics
- User action tracking

Read more about the API, features, and getting started in the [README](https://github.com/smaldd14/onboard-flow/blob/main/README.md).

## Use Cases

While I originally built this to onboard venues onto [Promoted](https://promoted.club)—sending introduction emails followed by case studies and feature explanations over 3 weeks—the system is flexible enough for various email campaign types:

- **Onboarding flows** - Welcome new users and guide them through your product
- **Reactivation campaigns** - Re-engage dormant users
- **Cold email campaigns** - Nurture leads through a sales funnel  
- **Educational sequences** - Deliver courses or tutorials over time
- **Event-driven campaigns** - Trigger based on user actions

## Getting Started

The application provides REST APIs for managing templates and sequences:
- Use `/templates` to create and manage email templates with dynamic placeholders
- Use `/sequences` to define email sequences with timing and conditions
- The app includes default templates and sequences to get you started quickly

### Deployment Options

For running Temporal, you have several options:

1. **Local Development**: Use the [Temporal CLI](https://docs.temporal.io/cli) for testing
2. **Production**: Choose between [Temporal Cloud](https://temporal.io/cloud) (managed) or [self-hosting](https://docs.temporal.io/self-hosted-guide)

**Important**: Don't rely on the Temporal CLI for long-running production workflows. Use either Temporal Cloud or a self-hosted Temporal cluster for reliability.

For detailed setup instructions, see the [project README](https://github.com/smaldd14/onboard-flow/blob/main/README.md).

