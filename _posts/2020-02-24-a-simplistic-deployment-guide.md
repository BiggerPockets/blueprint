---
author: Andy Wooten
title: A Simplistic Walkthrough of Deployments
description: Happy Reference for When Questions Arise
excerpt: This is a basic introduction to how deployment of production code is structured with some tips on how you can be proactive to make your life easier.
tags: engineering product
image: /assets/images/blank-branding-identity-business-cards-6372.jpg
---
## So, You Want to Deploy? "Priority", you say?! Well, let's do this!
I've gotten to speak about our deployment process a few times. It's generally only delivered in piecemeal chunks to resolve or explain a problem that's occurred. I thought I'd take the opportunity to run you through a chronological order of operations of our deployment process.

This isn't going to be a fully "under the hood" conversation; rather, it'll include good practices to keep life from turning into an occupied spiders nest of problems. I'm directing this post more towards the engineering side of our team; however, I think product can also benefit perhaps from having a better understanding of the alchemy we engage in from time-to-time. Here we go!

## Figure Out What You're Deploying
That one ticket you've merged in is good to go. That's great, and I'm excited that you're excited to click our "Deploy" button. Before we make our way over to Heroku-land, let's hang out at GitHub for a bit: [Recently Closed Pull Requests](https://github.com/BiggerPockets/biggerpockets/pulls?q=is%3Apr+is%3Aclosed+sort%3Aupdated-desc). JIRA is a great resource, but I, in my paranoid QA world, have little faith in the parity of features living in the merged lane with what has been merged into master.

That link is a great way to look over work that's on deck for deployment to production. You're going to want to do more here than read PR titles. Give a glance to the [Activity Tab](https://dashboard.heroku.com/apps/biggerpockets/activity). When was the last time we did a deployment to production? What's been merged since that we'll want to get familiar with. Generally speaking, as I'm scrolling through PR contents I'm looking at two things specifically: table migrations and data migrations. These can generate probably our worst kind of problems, up to and including deployment failure due to timeout. Keep in mind, we're just getting oriented; there's no need to pour through all of this extensively.

In the event there's a problem after a deployment succeeds, we'll save valuable time here by being able to remember who was doing what in which portion of the codebase. This isn't meant to be a blame game; rather, it's getting the most qualified resource to the most pressing problem as rapidly as possible. If you aren't familiar with the 7 Ps, here's the acronym: Proper Prior Planning Prevents Piss Poor Performance. Bad happens, but it happens more frequently and lasts longer if you don't forward plan for it.

## Open Sentry
[Sentry](https://sentry.io/organizations/biggerpockets/releases/?project=1818700) is your friend. Take a minute, just right quick, to familiarize yourself with the error which are already coming through. We've got some minor stuff which will live around most days. Errors/bugs are a part of the life cycle. They happen and we deal with them as we can. Rest assured, if you inadvertently trigger a major error, you will hear about it some where between the litany of forthcoming messages in #dev-questions, your team channels, and #sentry-integration. If you're a bit anxious about clicking the button, that's fine and relatively appropriate. Comfort comes from practice/time/failure. To avoid a person called wolf situation post-deployment, orient yourself to the errors which are currently trickling in. I'm a big fan of the 'Releases' tab for this reason. If you generate a new error, you'll be aware, and if it's happening with some great level of frequency, that's cool, we'll just have to see about fixing it or getting it fixed.

## Can we click the button now?
Yep! Let's get on with it. Drop an :underconstruction in #engineering team; this was declared as the official sign of deployment some time ago, as typing "Deploying" was becoming boring; this is a courteous heads up to others with horses in this specific race to watch for their features. We've done our prep work and due diligence, maybe more than you think is necessary, but best to overdue it in this case. You're going to want to navigate to our [Deploy tab](https://dashboard.heroku.com/apps/biggerpockets/deploy/github), scroll to the bottom page to the 'Manual deploy' section, and click the ominous looking 'Deploy Branch' button. Go time!

## What's going on here?
Alright, so we've kicked off the 'build' step. Exciting stuff. If you look at the logs that have appeared in the tiny modal on the screen there, you'll probably see some familiar looking things happening. The build step is the longest part of this process. It's a bit of a fluctuating pain point that I'm aware of. So, what's happening in our build step?

Without getting terribly gritty, we're installing dependencies and compiling our assets. Asset compilation is the time-eater. I'm hoping to move this to store our assets off-site soon to save time here. This step, barring a profoundly odd problem, should never fail. In order for it to fail, we'd most likely be looking at a syntax error in an asset file  or potentially one of our dependencies has fallen into the ether or was inappropriately referenced.

So, again, the build step hasn't failed terribly often for us in history, and if it does, that's great because we haven't done anything to our database yet, so user experience isn't being affected. 

## The Release Phase
The release phase is where our more dangerous stuff happens. Here, we're executing table and data migrations against our production databases. Potential for problems: relatively low with our peer review and QA process, but these have caused far more headaches than anything in our build process.

You can look at the logs for this as they kick off from the Deploy tab we were at earlier, or you can go to the Activity tab to see it when it pops up. Heroku understands our desired release task from our Procfile. We shoved this back into a super-handy rake task in `release_tasks.rake`. In our production environment, we will be running `rake db:migrate` and `rake data:migrate`, in that order.

Why the danger? Well, a number of terrible things can happen here unfortunately. If any of the migrations run partially then fail or timeout, we're now going to be living in very interesting world with partially mutated data. When these break, we enter a state where columns/instance may no longer align with what the app running in production expects. The new version of the application we just built will not reach users if there's a failure here.

This can be fairly benign in the case of a new table, or it can crash the forums. All depends on the nature of the change and the failure. I strongly encourage familiarization with migrations for this reason. Should they fail? No, absolutely not. Will they? Yeah, probably from time-to-time. Fighting complacency here is key.

## Tying it off
Don't rush it here. Generally after we see the release tasks complete successfully, you're going to be waiting for between 3 to 7 minutes as traffic is routed to our new application. It is possible, particularly with data migrations, to see transient errors during this time which self-resolve when the updated view which is looking for that renamed thing is made available to the user, for example. 

What are we doing during this time? We're looking at Sentry, specifically the related release that just popped up in the releases tab. We're monitoring the #sentry-integration channel, and to be honest, we're watching the clock tick a bit and maybe dealing with a minor task while we wait. 

Everything there? No errors getting thrown around? No complaints in Slack anywhere? Your new feature is behaving? Super-exciting stuff! You've deployed successfully and didn't break anything. Repeatability in the deployment routine is for the best. Deviating around because of complacency or ego is a good way to drag people into bad place for a decent amount of time. No fun, don't do that.

## Overview
  * Know what you're deploy: GitHub is your friend here
  * Leverage multiple channels for monitoring for errors: Sentry web page, Sentry Slack integration, and Heroku logs
  * Let people know you're deploying
  * Click the scary button
  * Watch your build phase
  * Watch your release phase
  * Wait 3 - 7 minutes for new features to be live
  * Monitor and triage debugging activities

# Ominous Music..
So, something, or several, broke. Well, that's no fun. If you were following along step-for-step, then you've kinda got the idea of where we're going from here. We've got a pretty short decision tree to traverse right quick. 

  * Find the PR responsible for the work. If it's multiple then reference both.

Is it a test? Turn it off or find someone who can. We recently introduced caching to our Optimizely stack [here](https://github.com/BiggerPockets/biggerpockets/pull/9435), with a follow-up task to clear it [here](https://github.com/BiggerPockets/biggerpockets/pull/9437).

Is it a data issue? This will happen from time-to-time; production data is not great sometimes and we'll have to quickly resolve some data problems. If you're here, the deployment/feature introduction have gone fine. We can't account for every bit of bad data from the past ever.

Can you fix it? Sometimes these things are just bad luck or a simple oversight. It happens. If you can patch it, open a PR, flag down #engineering-team about the issue to get eyes on the fix, referencing the Sentry error in your PR to help give context. CI runs clean, awesome. Re-deploy

Get the original author(s) to look into it. Is it a fast patch from their perspective? If yes, awesome. Stick with the "you can fix it" pathway, and we're done.

Original authors say something akin to "this is gonna take a bit". Well, this stinks. Now we're in for some real fun. You've got two options from here: revert/redeploy or rollback. There's benefits/drawbacks to both pathways. Quick summary:

Revert/redeploy: If you scroll to the bottom of the PR, you'll find a nifty button next to the "merged commit" message: "Revert". <- Congrats, this is what you want. Click the button. This will trigger a revert PR which will negate all of the changes made in the code base by the pull request. Sick right? Merge this in. We've now generated a state where the problematic work has been negated, and the rest of the work deployed is still viable in production. Click the deploy button. Barring any extraneous/renamed columns, you should be done. Post-deployment, emulate the experience to the best of your ability and watch Sentry to ensure the error has been resolved. Caveats to this approach: If multiple edits were made to the same files across PRs that deployed simultaneously, this can block GitHubs ability to perform a clean reversion. This is also incredibly slow; in addition to the time spent on the tree above, we have now kicked off another deployment which is going to take 12-15 minutes(on the high average end) to reach users and resolve the issue transiently. If migrations are involved, it's possibly you'll have to manually roll them back or undo a data migration to return production to a previous functional state.

Rollback: If you visit the [Activity Tab](https://dashboard.heroku.com/apps/biggerpockets/activity) again, we'll notice some handy buttons. You should see your most recent release `andy@biggerpockets.com Deployed 3802ead6 Today at 12:15pm`(for instance). If you look to the next previous release message, you'll find our rollback link: "Roll back to here". Heroku is kind enough to store our app slugs, complete with compiled assets and dependencies and whatnot. If you click this link, you will trigger an immediate release; it will not perform a build process, it will jump straight to the release phase(db:migrate/data:migrate) 
for that merge commit, and then roughly 3-7 minutes until this is to all of your users. Caveats: This is a super heavyweight move and will yank out a lot of work depending on the deployment; this will not automatically trigger down migrations(if those were even written). 

How do you make this decision? Well, it's partially a gut-check on how severe the error is combined with how complex the underlying data/table issues are that have been generated. I can't give a lot of guidance here. I prefer to revert and preserve the other work that's been deployed if it's a minor issue we're facing(define minor right?). If you're dealing with a catastrophic failure of, say, the forums, I'd prefer to rollback and do a subsequent deployment with the problematic work removed. I view it as a balance between user downtime,new feature introductions, and high-use feature availability; the harmonics there are a bit difficult to qualify. It's a pretty delicate balance, particularly when you're potentially dealing with multiple overlapping feature sets that have all mutated the database in some way. This kind of complexity can be fun, but it can also be nightmarish if you don't attack it in a structured manner.

## Conclusion
  * We've covered:
    - Steps to Deployment
    - A Basic Decision Tree for dealing with problems.
    - Benefits and caveats of the revert/redeploy versus the rollback method

In summary, it's only really as complicated as you let it become. If you blindly click the 'Deploy' button, you're lining yourself up(more likely the rest of the team) for a potentially giant headache; bad times. Take the proper steps to get oriented prior to the event, and you'll have a much better time, even in the event of a problem. Keep calm, do your diligence, and trust the team to be there if things are going squirrel-y. You got this!
