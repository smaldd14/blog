# Creating an AI-powered Blog Agent using TemporalIO

## Introduction

As a developer passionate about automation, I've been exploring TemporalIO to create workflows that streamline various tasks. Temporal is an excellent tool for developers who appreciate visualizing workflow steps directly in their code, building fault-tolerant processes, and focusing on the "happy path" while handling retries effortlessly. If you're unfamiliar with Temporal, you can learn more about its capabilities in their [official documentation](https://docs.temporal.io/docs/overview/what-is-temporal).

## Project Goal

My primary objective was to develop a straightforward workflow that encompasses the following activities (or steps, for those new to Temporal terminology):

1. Generate a blog outline based on a list of keywords
2. Create a blog post using the outline and keyword list
3. Save the blog post to a local file
4. Store the blog post in a database

While this initial setup is relatively simple, I have plans to expand the workflow with additional activities. Future enhancements may include:

- Sanitizing LLM responses
- Conducting keyword competition research
- Automatically posting the blog to a website
- Indexing the published page

For those eager to see the end result, here's a video demonstration of the workflow in action:

<video src="../assets/blog-agent-temporal.mp4" controls></video>

If you're interested in learning more about the implementation process, continue reading!

## Setting up Dependencies

To begin, I used [Spring Initializr](https://start.spring.io/) to create a new Spring Boot project with PostgreSQL and web capabilities. After downloading the project, I installed the following essential dependencies:

1. [Spring AI](https://docs.spring.io/spring-ai/reference/getting-started.html#repositories) repositories and bill of materials (BOM)
2. The latest [Temporal Java SDK](https://central.sonatype.com/artifact/io.temporal/temporal-sdk?smo=true)
3. [Anthropic Spring Boot starter](https://docs.spring.io/spring-ai/reference/api/chat/anthropic-chat.html#_auto_configuration)
4. [OpenAI Spring Boot starter](https://docs.spring.io/spring-ai/reference/api/chat/openai-chat.html#_auto_configuration)

Initially, I planned to use Anthropic's API for generating blog posts and outlines due to its superior results. However, at the time of writing, there was a default cap on the max-token output (4096) from the API. I've filed an [issue](https://github.com/spring-projects/spring-ai/issues/1068) to address this limitation, which will hopefully be resolved soon.

Given this constraint, I opted to use the reliable OpenAI GPT-4 model for generating outlines and posts.

## Implementing the Workflow

With the dependencies in place, I began setting up the Temporal classes to run the workflow.

**Note: To run Temporal, you need to connect to a Temporal server. You can set up a local Temporal server by following the [instructions in their GitHub repository](https://github.com/temporalio/temporal?tab=readme-ov-file#download-and-start-temporal-server-locally).**

### BlogPostStarter Class

The `BlogPostStarter` class is responsible for initiating the Temporal workflow:

```java
@Component
public class BlogPostStarterImpl implements BlogPostStarter {
    private final WorkflowClient client;

    public BlogPostStarterImpl(WorkflowClient client) {
        this.client = client;
    }

    @Override
    public void startBlogPostWorkflow(List<String> keywords) {
        String uuid = UUID.randomUUID().toString();
        WorkflowExecution wf;
        var stub = newWorkflowStub(BlogPostWorkflow.class,
                                   BlogPostWorkflowImpl.TASK_QUEUE,
                                   uuid);
        wf = WorkflowClient.start(stub::runWorkflow, keywords);
    }
}
```

### BlogPostWorkflow Class

The `BlogPostWorkflow` class orchestrates the activities, exemplifying Temporal's "workflow as code" philosophy:

```java
@WorkflowImpl(taskQueues = BlogPostWorkflowImpl.TASK_QUEUE)
public class BlogPostWorkflowImpl implements BlogPostWorkflow {
    public static final String TASK_QUEUE = "BlogPostWorkflow";
    private final BlogPostActivities activities;

    public BlogPostWorkflowImpl() {
        this.activities = Workflow.newActivityStub(BlogPostActivities.class,
                                                   ActivityOptions.newBuilder()
                                                           .setStartToCloseTimeout(
                                                                   Duration.ofHours(1))
                                                           .setHeartbeatTimeout(
                                                                   Duration.ofSeconds(600))
                                                           .build());
    }

    @Override
    public boolean runWorkflow(List<String> keywords) {
        String result = activities.generateOutline(keywords);
        GenArticleLlmResponse genArticleLlmResponse = activities.writePost(new OutlineKeywordPair(keywords, result));
        activities.writePostToFile(genArticleLlmResponse);
//      TODO: activities.sanitizeLlmResponse(genArticleLlmResponse);
        return activities.uploadToDb(genArticleLlmResponse);
    }
}
```

Note: The heartbeat timeout is set to 600 seconds for demonstration purposes. In a production environment, you would typically use a shorter timeout and implement a heartbeater.

### BlogPostActivities Class

The `BlogPostActivities` class implements each activity:

```java
@Component
@ActivityImpl(taskQueues = BlogPostWorkflowImpl.TASK_QUEUE)
public class BlogPostActivitiesImpl implements BlogPostActivities {
   private final LlmApiMgr llmApiMgr;
    private final BlogPostMgr blogPostMgr;

    public BlogPostActivitiesImpl(LlmApiMgr anthropicApiMgr, BlogPostMgr blogPostMgr) {
        this.llmApiMgr = anthropicApiMgr;
        this.blogPostMgr = blogPostMgr;
    }
    @Override
    public String generateOutline(List<String> keywords) {
        return llmApiMgr.generateOutline(keywords);
    }

    @Override
    public GenArticleLlmResponse writePost(OutlineKeywordPair outlineKeywordPair) {
        GenArticleLlmResponse genArticleLlmResponse = llmApiMgr.writePost(outlineKeywordPair);
        return genArticleLlmResponse;
    }

    @Override
    public BlogPost sanitizeLlmResponse(GenArticleLlmResponse genArticleLlmResponse) {
        return null;
    }

    @Override
    public void writePostToFile(GenArticleLlmResponse genArticleLlmResponse) {
        saveBlogPost(genArticleLlmResponse.slug(), genArticleLlmResponse.post());
    }

    public static void saveBlogPost(String urlSlug, String blogPostContent) {
        String filename = urlSlug + ".md";
        Path filePath = Paths.get(filename);

        try {
            // Create the file if it doesn't exist
            if (!Files.exists(filePath)) {
                Files.createFile(filePath);
            }

            // Write the content to the file
            try (BufferedWriter writer = new BufferedWriter(new FileWriter(filePath.toFile()))) {
                writer.write(blogPostContent);
            }

            System.out.println("Blog post saved successfully to: " + filePath.toAbsolutePath());
        } catch (IOException e) {
            System.err.println("Error saving blog post: " + e.getMessage());
            e.printStackTrace();
        }
    }

    @Override
    public boolean uploadToDb(GenArticleLlmResponse genArticleLlmResponse) {
        BlogPost blogPost = blogPostMgr.createBlogPost(genArticleLlmResponse.post(),
                                                       genArticleLlmResponse.slug(),
                                                       genArticleLlmResponse.keywords(),
                                                       genArticleLlmResponse.metaDescription());
        return blogPost != null;
    }
}
```

## Interacting with the LLM

The `LlmApiMgr` class manages interactions with either GPT-4 or Anthropic's Claude-3.5-Sonnet model. It uses Spring AI's `ChatModel`s and Java's `ResourceLoader`, `Resource`, and `StringTemplate` files for efficient LLM model calls:

```java
public class LlmApiMgrImpl implements LlmApiMgr {
    private final AnthropicChatModel anthropicChatModel;
    private final OpenAiChatModel openAiChatModel;
    private final Resource generateOutlineResource;
    private final Resource generateArticleResource;

    public LlmApiMgrImpl(AnthropicChatModel anthropicChatModel,
                         OpenAiChatModel openAiChatModel,
                         ResourceLoader resourceLoader) {
        this.anthropicChatModel = anthropicChatModel;
        this.openAiChatModel = openAiChatModel;
        generateOutlineResource = resourceLoader.getResource("classpath:/prompts/generate-outline.st");
        generateArticleResource = resourceLoader.getResource("classpath:/prompts/generate-article.st");
    }

    @Override
    public String generateOutline(List<String> keywords) {
        PromptTemplate promptTemplate = new PromptTemplate(generateOutlineResource);
        Prompt prompt = promptTemplate.create(Map.of("KEYWORDS", keywords,
                                                     "TARGET_KEYWORD", keywords.getFirst()));
//        Generation generation = anthropicChatModel.call(prompt).getResult();
        Generation generation = openAiChatModel.call(prompt).getResult();
        String description = generation.getOutput().getContent();
        return description;
    }
}
```

## Starting the Workflow

To initiate the workflow, I created a simple `@RestController` called `BlogController`:

```java
@RestController
public class BlogController {
    private final BlogPostStarter blogPostStarter;

    public BlogController(BlogPostStarter blogPostStarter) {
        this.blogPostStarter = blogPostStarter;
    }

    @GetMapping("/blog")
    public void startBlogPostWorkflow() {
        blogPostStarter.startBlogPostWorkflow(List.of("Google ads for apartments"));
    }
}
```

For the sake of this tutorial, I hardcoded the keyword list. In a production environment, you would likely pass the keywords as a request parameter, or in the request body for a POST call.

## Conclusion

In this project, we leveraged Temporal to create a robust workflow that:

1. Generates an SEO-optimized outline
2. Uses the outline to create an SEO-optimized blog post
3. Saves the blog post locally in markdown format
4. Uploads the blog post and SEO metadata to a database for future use

This automation streamlines the content creation process, ensuring consistency and efficiency in blog post generation. You can see the workflow in action by referring to the video at the beginning of this post.

If you're interested in exploring the full codebase or discussing this project further, feel free to reach out to me on [Twitter](https://twitter.com/devinsmaldore) or [LinkedIn](https://www.linkedin.com/in/devinsmaldore). I'm always excited to connect with fellow developers and discuss innovative ways to leverage AI and automation in our projects, especially when it comes to Temporal!