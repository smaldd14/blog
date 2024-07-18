# Creating an AI powered blog agent using TemporalIO

## Introduction
I've been messing around with TemporalIO for awhile now, creating many different 
workflows to automate tasks that I don't really want to be doing. Temporal is great for 
developers who like to see the workflow steps right from the code, create workflows
that are fault tolerant, handle retries, and mainly worry about the "happy path".
You can read more about Temporal [here](https://docs.temporal.io/docs/overview/what-is-temporal).

## Goal
The goal of this initial blog agent setup was to create a simple workflow that would have
the following activities (steps for those not familiar with Temporal lingo):
1. generate a blog outline given a list of keywords
2. generate a blog post given the outline, and keywords list
3. save the blog post to a database

This was a simple enough workflow to get setup, but I plan to add more activities as I build this out.
Steps such as *sanitizing LLM response, keyword competition research, posting the blog to a website, and indexing
the posted page* are a few of the next steps I am planning on taking.

## Setting up Dependencies
I started off by going over to [Spring Initializr](https://start.spring.io/) and
creating a new Spring Boot project with postgresql and web capabilities.

Once I downloaded the new project, next I had to install the following dependencies:
* [Spring AI's](https://docs.spring.io/spring-ai/reference/getting-started.html#repositories)
documentation to bring in the necessary repositories, and bill of materials (BOM). 
* Latest temporal java-sdk version (can be found [here](https://central.sonatype.com/artifact/io.temporal/temporal-sdk?smo=true)).
* [Anthropic Spring boot starter](https://docs.spring.io/spring-ai/reference/api/chat/anthropic-chat.html#_auto_configuration)
* [OpenAI Spring boot starter](https://docs.spring.io/spring-ai/reference/api/chat/openai-chat.html#_auto_configuration)

Ideally, I would like to use Anthropic's API to generate the blog post and outline because I 
have found that I get better results. But as of the time of my writing, there was a 
default cap on the max-token output that the API would give back (4096). I filed an [issue](https://github.com/spring-projects/spring-ai/issues/1068)
and hopefully that will be fixed soon.

So, needless to say, I went ahead with the trusty OpenAI GPT-4o model to generate the outline and post.

## Setting up the Workflow
Once I had all of my dependencies installed, I began setting up the Temporal classes to 
run a workflow.

**Before we do dive in, it is worth mentioning that in order to run Temporal, you need to connect
to a Temporal server. You can run a local Temporal server, by following [these directions](https://github.com/temporalio/temporal?tab=readme-ov-file#download-and-start-temporal-server-locally)**

The first class I setup was `BlogPostStarter`. This class handles starting the Temporal workflow.
It was quite simple, and looked like this:
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

Next, the `BlogPostWorkflow` class handles the orchestrating of activities. This is where 
Temporal's "workflow as code" mantra comes about. Developers can easily understand the
order of activities and what is running when.
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
        GenArticleLlmResponse genArticleLlmResponse = activities.writePost(
                new OutlineKeywordPair(keywords, result));
//      TODO: activities.sanitizeLlmResponse(genArticleLlmResponse);
        return activities.uploadToDb(genArticleLlmResponse);
    }
}
```

It is worth noting that I set the heartbeat timeout to 600 seconds. Normally you would not do this in a production environment, but for the sake of brevity, I set it to 600 seconds.

And then, the `BlogPostActivities` class handles the implementation of each activity:
```java
@Component
@ActivityImpl(taskQueues = BlogPostWorkflowImpl.TASK_QUEUE)
public class BlogPostActivitiesImpl implements BlogPostActivities {
    private final AnthropicApiMgr anthropicApiMgr;
    private final BlogPostMgr blogPostMgr;

    public BlogPostActivitiesImpl(AnthropicApiMgr anthropicApiMgr, BlogPostMgr blogPostMgr) {
        this.anthropicApiMgr = anthropicApiMgr;
        this.blogPostMgr = blogPostMgr;
    }

    @Override
    public String generateOutline(List<String> keywords) {
        return anthropicApiMgr.generateOutline(keywords);
    }

    @Override
    public GenArticleLlmResponse writePost(OutlineKeywordPair outlineKeywordPair) {
        GenArticleLlmResponse genArticleLlmResponse = anthropicApiMgr.writePost(outlineKeywordPair);
        return genArticleLlmResponse;
    }

    @Override
    public BlogPost sanitizeLlmResponse(GenArticleLlmResponse genArticleLlmResponse) {
        return null;
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

Lastly, I created a quick @RestController called `BlogController` to start the workflow:
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

Finally, I got this all running, used POSTman to hit the `/blog` endpoint, and saw the workflow do its thing! Here's a video of the workflow in action:

<video src="/assets/blog-agent-temporal.mp4" controls></video>

