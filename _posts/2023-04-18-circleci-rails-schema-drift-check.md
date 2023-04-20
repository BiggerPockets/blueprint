---
author: Aaron Breckenridge and Rajaa Boulassouak
title: Checking for Rails schema drift in CircleCI
description: Schema drift is a common problem that teams encounter as their teams get larger and the pace picks up. This article describes how we tackled that at BiggerPockets.
tags: engineering rails
image: /assets/images/dunes.jpg
---

Schema drifts happen when your local development environment stops matching the production environment. These occur
naturally as engineers simultaneously work on modifications that affect the same database entities. For example, if two
engineers both write a migration that adds a column to a table, they may end up in a situation where their databases
have the columns in different order. This causes problems whenever the two engineers check in subsequent migrations:
each of their versions of `db/schema.rb` are different because the migrations were run in a different order.

Schema drifts can also happen when people make emergency changes to a production schema without backporting those
changes into the `db/schema.rb` file via a migration.

Here at BiggerPockets, we've started to run into this problem as the pace of development has picked up. To address
this common problem, we wrote and installed a CircleCI configuration that runs a task to compare each PR's schema
against what's currently checked in to main. If the branch includes a migration, the task loads whatever the main
branch's schema is, runs the branch's migrations, and then compares the resulting schema to what's being proposed in
the Pull Request. If there's a difference, the schema has drifted.

Here's an abbreviated version of the task. Note that it doesn't include some necessary steps for installing dependencies
since that will depend on how you've configured your pipeline.

```
# in .circleci/config.yml
jobs:
  compare-migration:
    executor: default-rails-setup
    parallelism: 1
    steps:
      - clone-repo
      - run:
          name: Checkout the base branch and restore base schema
          command: |
            git fetch origin main
            git checkout main
            git reset --hard origin/main
      - store_artifacts:
          path: db/schema.rb
          destination: schema-main.rb
      - run:
          name: Reload the schema from main
          command: |
            bundle exec rake db:drop db:create db:schema:load
      - run:
          name: Checkout PR branch
          command: |
            git checkout $CIRCLE_BRANCH
      - store_artifacts:
          path: db/schema.rb
          destination: schema-branch.rb
      - run:
          name: Compare PR schema.rb to migrated schema.rb
          command: |
            mv db/schema.rb db/schema_from_pr.rb
            bundle exec rails db:migrate
            diff -U 3 db/schema_from_pr.rb db/schema.rb
      - store_artifacts:
          path: db/schema.rb
          destination: schema-after-migration.rb
```

If there are any differences, they'll be reported via the `diff` command. This job also generates three artifacts so
that the schemas can be compared at the different stages:

1. The current schema from the main branch (`schema-main.rb`) before any migrations have been run.
1. The checked-in schema from the branch (`schema-branch.rb`).
1. The schema after running the branch's migrations against the schema loaded from the main branch
   (`schema-after-migration.rb`).

Essentially, the `schema-branch.rb` and `schema-after-migration.rb` file need to match, or the schema will start to
drift.
