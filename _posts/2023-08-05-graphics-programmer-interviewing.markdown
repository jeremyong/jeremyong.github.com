---
layout: post
title: Interviewing Graphics Programmers
date: 2023-08-05 00:00
categories:
- Graphics
- Interviewing
---

This is a light brain dump intended to describe the interviewing process as it pertains to graphics programmers (including prospective ones).
If you are in the business of assessing other graphics programmers in an interview, you'll likely find that getting a good signal-to-noise ratio
can be quite tricky. Things don't get much easier on the other side of the table for prospective graphics programmers looking
to get hired either. Having both been interviewed a number of times and also having conducted many more interviews for graphics programming candidates,
I decided to compile some notes I've accumulated over the years. If you're an experienced graphics interviewer yourself, feel free to send feedback
or comment via the Mastodon account linked in the site header above.

Note that most of this post is written from the perspective of an interviewer conducting interviews, but the information should be equally useful for
someone preparing for an interview. At the very bottom, you'll also find sections of specific advice targeted at both the interviewer and interviewee respectively.

## The broad strokes

Broadly speaking, there are several (what I'm calling) personas within the graphics programming world.
It's unlikely you'll find an individual perfectly matching the description of one persona or another, but the taxonomy is still helpful. If an individual
overlaps several personas, you can simply tailor the questions accordingly.

- RHI programmer
- Tools and asset pipeline programmer
- Technique specialist
- Generalist

Let's look at each of these personas in turn along with some questions I've accumulated for each persona over the years. I will be focusing here mainly
on the technical aspects, but please do supplement this with an evaluation of soft skills that I believe are just as critical.

### RHI programmer

The RHI programmer is concerned chiefly with two tasks: 1. feeding data to the GPU and 2. recording and submitting commands to the GPU.
Of all the personas, this is the most platform-specific task because different GPUs and platforms interoperate with the host program differently.
Good RHI programmers tend to have extensive experience with at least two platforms/APIs (needed to understand the degrees of freedom in an RHI abstraction),
but of course, this experience can certainly be attained on the job (especially so for console APIs). Some potentially relevant questions:

1. Describe the different ways one could expose buffer data to a shader, and their pros and cons.
2. What is typically meant by the term "texture layout" (aka "image layout") and why is it significant?
3. What are some memory hazards you can think of that a good renderer would avoid?
4. What is a "descriptor" and what are some strategies you've seen for managing them?
5. What is a "pipeline state object" and what are some strategies you've seen for managing them?
6. Suppose you are told that the game hitches poorly at a particular point in the level. Can you describe your approach to diagnosing the issue?

Beyond this, you could imagine asking a number of questions that might touch on more specific expertise based on prior experience. For example:

- Texture streaming systems
- Render graph architecture and automatic barrier placement
- Shader compilation pipelines
- Swapchains and frame pacing
- Techniques for reducing input latency
- Instrumentation for measuring performance or diagnosing TDRs

and the list goes on. In fact, all interview sessions should be adapted at least in part based on prior candidate experience where possible.

### Tools and asset pipeline programmer

If the customer of the RHI programmer is the IHV and other graphics programmers, the customer of the tools and asset pipeline programmer is the artist or technical artist.
After a renderer's pipeline is well-defined and a set of asset types are ossified, it's important to create a set of tools that enable artists to
be as productive as possible in producing and validating those assets. Depending on the project, source asset sizes could easily exceed the terabyte range,
so the importance of an efficient art pipeline cannot be understated.

I've found that interviewing for this engineer is highly dependent on the setup at your studio. Here are some considerations before determining how to assess the candidate:

- How are tools in your studio authored? Qt? Engine-native? WPF/MFC/Win32? Something else?
- Do you rely on distributed caching?
- Is your infrastructure on-prem? In the cloud? A mixture of both?
- What DCC tools are supported?
- What asset formats are in use for textures, scenes, meshes, etc.? An off-the-shelf format? Proprietary formats? Both?

All of the above choices constitute a different set of skills that may or may not be relevent to the job. For example, perhaps your pipeline heavily leans on
a particular cloud provider, in which case familiarity with that cloud provider (or the cloud in general) might be a huge plus and a good thing to interview about.
The idea is not to quiz the candidate on all the particulars of your setup, but to get a rough sense for whether the candidate could get up to speed in a reasonable time frame,
and be productive in a medium-to-long time horizon.

### Technique specialist

Many larger studios may have several roles open for graphics programmers that specialize in a particular technique. Some examples include:

- Volumetric rendering
- Terrain
- Atmospheric effects
- Particles
- Anti-aliasing
- Shadows
- Color treatment
- Global illumination

and so on. Interviewing for such roles can be a bit more daunting given that unless the candidate happens to specialize in _your_ particular specialization,
it's expected that they possess significantly more subject matter expertise than you do. I personally like to remind myself that the goal of the interview isn't
to convince everyone that I'm smarter or know more than the candidate. As with all the other roles (and especially this one), the goal of the interview is to
identify relevant strengths and weaknesses of the candidate with as little bias as possible.

One thing I've found is that it's quite possible to have a very productive conversation with a person who is well-versed in a topic, provided you have a minimal
prerequisite background and actively listen. After all, a university professor can conduct productive conversations with students. By "minimal prerequisite background," what I mean is
that for a given topic, say, global illumination, it's important to understand:

1. Why the given problem is hard.
2. How your current studio generally approaches the problem.
3. Some common solutions to the problem.

To this end, I find myself occasionally brushing up on topics prior to an interview if I believe it's necessary. Another option is to have similar specialist
conduct this portion of the interview. If this is an option, great, but note that I've found that this paradoxically doesn't necessarily mean the interview is more productive.

Assuming you can navigate the "gist" of an area of specialization based on the topics above, you should be able to talk at length with the individual about
prior solutions they have worked on. For each solution discussed, I recommend keeping the discussion _critical_. Each choice came with a set of tradeoffs.
What were those tradeoffs? Why did it work for your constraints or art pipeline? What other alternatives were considered? Why were they rejected? Continuing
in this way, I've found that excellent candidates tend to be _idea generators_. They know the space like the back of their hand, and exhausting the limits
of the depth of their knowledge in that area in one to two hours is impractical -- this is a good sign.

If the discussion appears to wane, remember that it's OK to ask "simpler" questions about how things work. Not every question has to be research grade!
I have been surprised by the results in one of two ways here. First, I've encountered candidates that take a simple question and in the ensuing discussion,
point out things I had not considered before (great!). Second, I've encountered candidates that, in spite of my assumptions, were not able to explain the
simpler concepts suitably. In both cases, I learned something valuable about the candidate.

### Generalist

Many candidates will fall in the "generalist" category, where they may productively work on tasks you'd assign to the other personas mentioned previously.
Here's a subset of some questions that I've asked as an interviewer (of varying difficulties and presented in no particular order):

1. What happens when you sample a texture in a shader?
2. Given two spheres, determine if they intersect
3. What is meant by shader occupancy, and what affects occupancy?
4. Author a shader that produces a checkerboard pattern
5. What is a BxDF and what are some requirements for a function to be a valid BxDF?
6. Is branching on a GPU slow? If it depends, what does it depend on?
7. What is a "Moiré pattern" and why might it show up?
8. Why is a z-prepass useful? When would it not be useful?
9. What is "gamma"?
10. Your "hello triangle" fails to produce a triangle. What are some problems you'd anticipate?
11. What makes a GPU fast? What are some tradeoffs made in achieving that acceleration?

Not being able to answer one or more questions such as the ones above in no way immediately disqualifies a candidate.
However, a number of them transition nicely into broader topics that will ideally showcase aptitude. Most of the questions above
were tuned to be topics a candidate would likely encounter within the first few years of graphics programming, but there are bound to be
areas where a candidate might have missed entirely, or areas where the candidate possess a higher degree of depth than average.

## General advice for interviewers

As with general interviewing, achieving a good signal is everything. If you spend all your time trying to delve into a very specific topic, only
to find the candidate struggled from start to finish, you may have missed valuable insights in other aspects where the candidate would have demonstrated value.
Similarly, if you hopped continuously from topic to topic without ever diving deep, you may misattribute a candidate as an excellent hire, only to
find out later that the candidate lacks depth in key areas.

The interview must be _consciously directed_ to balance between:

1. The job requirements
2. The candidate's prior experience

To the extent that items one and two overlap, the interview should be relatively straightforward. Otherwise, you will need to extrapolate a solid signal
based on the candidate's expertise to infer if the candidate is a good fit for the job requirements. Don't fall into the trap of only sticking to
topics the candidate is expected to do well in though. Again, balance is key.

It goes without saying that all this advice, and indeed, this entire post should be _in addition_ to general interviewing guidelines (which
include guidelines for hiring a good software engineer in general).

If there was a challenge I have struggled with the most, it's one of calibration. It's not easy to come up with a set of "canned questions" we would
expect every competent graphics programmer to answer, while still accurately assessing requirements needed for every role. Ultimately, you will need
to rely on your intuition, which can leave room for bias to creep in. This is where having multiple opinions and weighing feedback from an ensemble
of individuals of varying backgrounds helps. That said, if you have ideas on this front, feel free to DM or comment on Mastodon!

## General advice for candidates

As seen above, interviewers in this field should not expect exhaustive knowledge of all things graphics. Good graphics programming interviews are,
in my opinion, somewhat flexible because the graphics programming is a broad discipline, and achieving a good signal from an interview demands that
flexibility. Prior to going into an interview, I highly recommend the following:

- Being clear about what you've worked on or studied
- Being clear about what you're interested in
- Reviewing your prior work and experiences so you can easily converse about the problems you've solved, and how you solved it

The first two items are mainly to aid the hiring manager understand how best to assess that you're a good fit for the role.
The last item is a good general interview preparation tip, especially so for graphics because talking about prior experience is sometimes the
only real way to evaluate a person's understanding of a topic.

If you've never had a graphics programming job before, that's ok! Graphics programming roles are hard to fill so many studios are willing to train
up graphics programmers provided sufficient aptitude is demonstrated. Demo projects are key here, so be prepared to have code to show (even if
its reviewed offline prior to the interview). Study job postings and try to identify what skills are needed for a role you are interested. In some
cases, you might even consider reaching out to the hiring manager for preliminary advice on how to prepare.
