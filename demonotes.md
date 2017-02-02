# Prep

> It's important to note that stackstorm has a fantastic web UI. We're using the CLI in this demo
> primarily for speed and familiarity, but know that this is also an option

First, ensure the following has been done:

- Do not disturb on laptop, everything unnecessary closed out
- Fresh "vagrant up" of `st2vagrant`
- Manual `sudo apt-get remove ansible` (lazy solution to a temp problem)
- Reconfigure twitter sensor to poll every 10 seconds
- Restart the VM

# Exploring Environment

Show Vagrantfile config, and provisioner output to audience

Log in to the VM with `vagrant ssh`

Show that stackstorm is running with:

- `st2 -h`
- `st2 --version`
- `st2ctl status`

# Actions

We can see all of the available actions using the `st2 action list` command:

	`st2 action list`

Run a simple action.

	st2 run core.local date

An "execution" is a record of an action being run. We can see the list of past executions with:

	st2 execution list

We can use the execution ID to get details about a specific execution:

	st2 execution get < insert ID here >

We can see how this action is constructed by navigating to the directory where it is stored:

	cd /opt/stackstorm/packs/core/actions

The metadata file for this action is here:

	cat local.yaml

Note that this runs as a local shell command, so no script file is needed for this action runner type.

Time for a more interesting example. What about posting a tweet?
Stackstorm can't natively post anything to Twitter, that particular functionality is provided by the "twitter" pack. So let's install that now.

	st2 pack install twitter

This command not only downloads and installs the packs, but also takes care of installing
dependencies if possible.

We can see it's installed in the list now:

	st2 pack list

This pack happens to have both sensors and actions that we can use in stackstorm.
We'll use one of the actions to post a sample tweet.

	st2 action list --pack=twitter

First, let's check out how the update_status action works. Notice this action runs as a
python script, so our YAML metadata file is accompanied by a Python script

(note arguments in metadata file)
	
	cd /opt/stackstorm/packs/twitter/actions
	cat update_status.yaml
	cat update_status.py

If we were to try to run this action now, it would fail, because the pack hasn't been configured.
Many packs require this step, for instance, to provide authentication options.

Note that this pack has an example configuration you can use to get started:

	cd /opt/stackstorm/packs/twitter/
	cat twitter.yaml.example

There's a built-in command for generating this automatically with a wizard:

	st2 pack config twitter

This would generate a configuration file for us, and place it in /opt/stackstorm/configs

However, I have a known working configuration file that already has all my secrets set up
so for this presentation, I'll just copy the config and reload:

	sudo cp ~/st2-intro/configs/twitter.yaml /opt/stackstorm/configs/
	st2ctl reload

Now that our pack is configured, we should be able to send a tweet:

	st2 run twitter.update_status status="Hello, world!"

# Sensors and Triggers

Actions are cool and all, but they don't promote event-driven automation on their own. For that, we need to understand Sensors.

Just like everything else, Sensors are distributed in packs. We can see what sensors came with the twitter pack here:

	st2 sensor list --pack=twitter

Remember that sensors just bring data into stackstorm. Triggers are what allows stackstorm to recognize that something interesting has happened with that data.

Triggers are defined within the sensors themselves. We can see what triggers are registered for this pack with a similar command:

	st2 trigger list --pack=twitter

We're going to use the search sensor, so let's look at how that's constructed:

	cd /opt/stackstorm/packs/twitter/sensors
	cat twitter_search_sensor.yaml
	cat twitter_search_sensor.py

# Rules

Rules are what tie everything together. With rules, we watch triggers for events, and take action accordingly.

	st2 rule list

You'll notice that the twitter pack also came with a predefined rule. Let's take a look at that:

	cd /opt/stackstorm/packs/twitter/rules
	cat relay_tweet_to_slack.yaml

As you can see, it's referencing the slack integration pack, but we haven't installed or configured that, so let's do that now.

	st2 pack install slack

Similar to the Twitter pack, we need to configure this pack too. We can see the example config that's distributed with the pack:

	cd /opt/stackstorm/packs/slack/
	cat slack.yaml.example

Again, I have no desire to share secret keys on a live stream so we'll just copy in my own config.

	sudo cp ~/st2-intro/configs/slack.yaml /opt/stackstorm/configs/
	st2ctl reload

Okay, this rule is ready to go, so tweet away! Use the hashtag #stackstormplug
(make sure to tweet from stackstorm_bot here at least once)

	st2 execution list --action=slack.post_message
	st2 execution get < insert execution ID >


# Workflows

Actions are great, but usually we want to do several things for remediation. For starters, we might want to kick off different actions based on some criteria.
In this example, we don't want to post to slack if it comes from the @stackstorm_bot account because we obviously already know about those tweets, we're making them.

We can do this with workflows. Stackstorm workflows are written in OpenStack Mistral (mistral is actually bundled with stackstorm if you use the one-line installer)

Just like any other action, workflow require a metadata file in order to be referenced by rules. Let's create that now:

	cd /opt/stackstorm/packs/slack/actions
	sudo cp ~/st2-intro/workflows/post_tweet_if_not_stackstorm_bot_meta.yaml .
	cat post_tweet_if_not_stackstorm_bot_meta.yaml

The actual workflow is contained in the referenced file "workflows/post_tweet_if_not_stackstorm_bot.yaml" so let's create that now:

	cd /opt/stackstorm/packs/slack
	sudo mkdir workflows
	cd workflows/
	sudo cp ~/st2-intro/workflows/post_tweet_if_not_stackstorm_bot.yaml .
	cat post_tweet_if_not_stackstorm_bot.yaml

> Some of the more advanced logic in Mistral workflows is written in YAQL (Yet Another Query Language): http://yaql.readthedocs.io/en/latest/
> Note that in the upcoming 2.2 release, Jinja2 support will also be present for things like this.

Note that, if the tweet comes from the @stackstorm_bot account, it won't post to slack, but instead will call an Ansible playbook:

	cat ~/st2-intro/ansible/hello_from_ansible.yaml

Of course, we need to install the Ansible pack for this to work, so let's do that now.

	st2 pack install ansible

Just like before, we can install our workflow using "st2 action create", referencing the workflow's metadata file:

	st2 action create ../actions/post_tweet_if_not_stackstorm_bot_meta.yaml
	st2 action list --pack=slack

Now that the workflow is in place, we want to use this workflow instead of our simple action.
So let's disable our existing rule and create another one that refers to our new workflow:

	st2 rule disable twitter.relay_tweet_to_slack
	cd /opt/stackstorm/packs/twitter/rules
	sudo cp ~/st2-intro/rules/relay_tweet_to_slack_conditional.yaml .
	cat relay_tweet_to_slack_conditional.yaml

We can install this rule like we did before. We should have only one enabled rule now in the Twitter pack:

	st2 rule create relay_tweet_to_slack_conditional.yaml
	st2 rule list --pack=twitter

As with before, we can watch for new executions with:

	st2 execution list

Note that because we're using a workflow, executions are actually hierarchical - the root execution is the original
call to our workflow, and each child execution is a different action taken inside that workflow.

	st2 execution get < Pass in various execution IDs here

# END OF DEMO
