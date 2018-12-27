# Puppet Slack

## Description

A Puppet report handler for sending notifications of Puppet runs to [slack](http://www.slack.com).

Includes auto-bootstrapping for your master(s), flexible hook routing, inclusion of logs by severity 
or tags, no-op run coloring and flagging, cached catalog failure marking, job ID inclusion if off schedule etc.

## Note to Users of this Report Processor
Slack messages are quite flexible in their formatting, and as such this report processor will by default use
message attachments for the message, rather than stuffing the data into the message text.

There is a maximum count of attachments that can be made onto a slack message - and any message posted to a
webhook with a greater than 100 attachments will be rejected by the API.

As such, the module is configurable (but enforces maximum bound) of attachments that can be put on a message.

If there are too many changes to report in attachments format 
(which appear more nicely on mobile clients especially, and allow for short 'fields' to be added), then this
report processor will switch to embedding of the change set within the primary attachment, 
var `event_data` in `slack.rb`.

You can make this fallback behavior ALWAYS occur, should you wish, by setting param `$slack_max_attach_count`
to 0, or setting `false` the init.pp class param `$slack_events_as_attach`.  We'll cover this later with the other options, but it deserves highlighting as it quite drastically
changes message formatting.

Be aware, if you leave it as its set by default, your messages will look nicer (in the author's opinion) -
specifically you'll avoid markdown not marking up correctly when in an attachment (even when content flagged)
on Android slack client (v2.7.0).  Iphone client, and desktop clients do not appear to encounter this issue.

There is also 'fallback' text for clients which cannot display messages nicely at all.  I have not encountered
such a slack client, and have not observed how the output looks.

## Requirements
You're Puppet Master needs ruby gem `slack-notifier ~> 2.3.2` available to puppetserver ruby 
('twiddle-wakka' is an absurd name, pessimistic operator is better).  

In Puppet Server 6 at the very least, including `init.pp` of this module will sort that out for you, but
if you experience issues they'll be logged specifically by the report processor, and you can install the gem
as so (I give it to both the server and client ruby because I find the whole way puppet does this awfully
confusing):

~~~~
sudo puppetserver gem install slack-notifier
sudo /opt/puppetlabs/puppet/bin/gem install slack-notifier
~~~~

The bootstrapping of this module and its config file maintenance required for registration of the report
processor requires (2.4.0 or higher in the initial release, did not test lower ver):

~~~~
#e.g. in Puppetfile
mod 'inifile',
    :git => 'https://github.com/puppetlabs/puppetlabs-inifile.git',
    :tag => '2.4.0'
~~~~

## Usage / Installation
1. create hiera data for your puppet masters for this report processor (you could specify it as a resource,
I won't cover that), see an example configuration in later sections.

2. include the class `report_slack` for puppet masters that should run this report processor.

## Basic Configuration
In the most basic sense, configure hiera data for your puppet master nodes that will be found on implicit lookup
of class params as follows:

*ENSURE YOU SPECIFY THE CORRECT USER AND GROUP FOR THE GENERATED REPORT PROCESSOR CONFIG FILE*

*YOU MUST ALSO DO THIS FOR THE PUPPET MASTER SERVICE NAME*

*THIS MODULE ASSUMES YOU ARE USING PUPPET ENTERPRISE, IF YOU ARE NOT, SPECIFY THE APPROPRIATE ACCOUNT/GROUP*

the example below shows basic config when using free puppet.
~~~
#<my hieradata module>/data/nodes/my_puppet_master.moogles.yaml
---
report_slack::slack_default_webhook: 'https://hooks.slack.com/EXAMPLE/YOUR/WEB/HOOK'
report_slack::owner: puppet
report_slack::group: puppet
report_slack::puppetmaster_service: puppetserver
~~~

include the class in your site.pp via hiera lookup / node definition for master(s) / however you do it.

By default, reporting will occur when the run is status `changed` or `failed`.

## Paramaters to `init.pp` and their Default Values Explained

### `String[1] $slack_default_webhook`
This is your default slack webhook, to which messages will be posted if there is no explicit routing
data relevant to the report `self.host` in the hash of hashes `$slack_routing_data`, or this optional 
param is `empty` or `undef`, or if an entry exists in `$slack_routing_data` for a regex pattern match on 
the `self.host` which DOES NOT include an inner hash attribute specifying a target webhook 
(i.e. you can specify e.g. this node reports on statuses different than others).
  
### `Boolean $enabled = true`
Set `true` and the report_slack reports processor will be enabled.
Set `false` and the registration of the reports processor will be ensured `absent`
  
### `Array[String] $slack_default_statuses  = [ 'failed', 'changed' ]`
An array of statuses that should prompt reporting to slack, by default, if no specific status information
is included for a regex match key in `$slack_routing_data`
For example, changing this value to `['failed', 'changed', 'unchanged']` would end up reporting for 
unchanged puppet runs as well.  This might lead to a lot of spam in your webhook channel, but may be
appropriate for particular nodes that you'd like to direct somewhere else.  To do that look into using
`$slack_routing_data`.
    
### `Array[String] $slack_attach_log_levels = ['warning', 'err', 'alert', 'emerg', 'crit']`
*GLOBAL OPTION ONLY IN THIS VERSION*

When a report is to be sent, what log levels should be included for the report?  If there exists log data
with this report that matches those log levels, it will be appended in a dedicated message attachment which
will format each message as markdown code blocks, including the severity, but excluding timestamp (as its 
all related to 'this run', and for brevity).

If you care about particular resources and seeing logging associated with that, instead of including more
verbose log level options here, consider tagging the resources, and using `$slack_attach_log_tags`

### `Array[String] $slack_time_metrics_keys = ['config_retrieval', 'total']`
*GLOBAL OPTION ONLY IN THIS VERSION*

An array of report 'time' metrics to include with the slack post as fields in the main message attachment.
The param `$slack_include_run_time_metrics` must be set `true` for these to be included.
This can be pretty much any resource type, as well as a number of in-built diagnostic / performance analysis
focused metrics.  For example you could add `['exec', 'package']` to the array, 
to see how much time was spent in those resource types.  This can be quite handy.

### `String $owner = 'pe-puppet'`
*It is absolutely essential* that you configure this value for owner as appropriate to your puppet 
server install.  See the example config for puppet-free in prior sections of this documentation (under 
'Basic Configuration').

### `String $group = 'pe-puppet'`
*It is absolutely essential* that you configure this value for group as appropriate to your puppet 
server install.  See the example config for puppet-free in prior sections of this documentation (under 
'Basic Configuration').

### `String $puppetmaster_service = 'pe-puppetserver'`
*It is absolutely essential* that you configure this value for puppet master server service name as 
appropriate to your puppet server install.  See the example config for puppet-free in prior sections of this
documentation (under 'Basic Configuration').

### `String $gem_provider = 'puppet_gem'`
*It is absolutely essential* that you configure this value for puppet gem provider name as 
appropriate to your puppet server install, if it varies from this default.  

### `String $slack_failed_color = 'danger'`
*GLOBAL OPTION ONLY IN THIS VERSION*

Slack names their message colors in some cases, this 'danger' color is merely a alias for a suitably scary
red, indicative of failure. 

You can replace this value with any hex code you like, e.g. `#FF0000`

Search google for 'color picker hex', and it'll auto-suggest and popup a color picker, or use something such
as: [htmlcolorcodes.com/color-picker](https://htmlcolorcodes.com/color-picker/)  - you want the 'hex' code.

### `String $slack_failed_emoji = ':fire:'`
*GLOBAL OPTION ONLY IN THIS VERSION*

Slack has emoji markup, where the name or alias in your workspace for the emoji 'this_emoji' is specified as
the name of the emoji surrounded by the ':' character. so e.g. `:slot_machine:`.

This emoji will appear as flair for at the top of your report messages with a failed status, 
in its text, and aids in visual highlighting of report status!

### `String $slack_changed_color = 'warning'`
*GLOBAL OPTION ONLY IN THIS VERSION*

Slack names their message colors in some cases, this 'warning' color is merely a alias for a yellowish hue,
that might evoke a sense of caution or awareness in the viewer.
 
You can replace this value with any hex code you like, e.g. `#FFF300`

### `String $slack_changed_emoji = ':warning:'`
Slack has emoji markup, where the name or alias in your workspace for the emoji 'this_emoji' is specified as
the name of the emoji surrounded by the ':' character. so e.g. `:slot_machine:`.

This emoji will appear as flair for at the top of your report messages with a changed status, 
in its text, and aids in visual highlighting of report status!

### `String $slack_unchanged_color = 'good'`
Slack names their message colors in some cases, this 'good' color is merely a alias for a lovely green,
that might evoke a sense of satisfaction or serenity in the viewer.
 
You can replace this value with any hex code you like, e.g. `#1ED505`

### `String $slack_noop_color = '#439FE0'`
*GLOBAL OPTION ONLY IN THIS VERSION*

It is often useful to know if a run report concerns a `no-op` status!

Messages with `self.noop` in the report data are no-op runs, and this report processor will flag them as so.
This parameter corresponds to the color of the message bar when the run is --noop.

Its default value is a passively aggressive moody sky blue, intended to pique the viewers interest, yet not
offend overtly.

You can replace this value with any hex code you like, e.g. `#BCD6E9`

### `String $slack_noop_emoji = ':shield:'`
*GLOBAL OPTION ONLY IN THIS VERSION*

Slack has emoji markup, where the name or alias in your workspace for the emoji 'this_emoji' is specified as
the name of the emoji surrounded by the ':' character. so e.g. `:slot_machine:`.

This emoji will appear as flair for at the top of your report messages with a self.noop of true, 
in its text, and aids in visual highlighting of report status!

### `String $slack_noop_event_color = '#48B0F9'`
*GLOBAL OPTION ONLY IN THIS VERSION*

As opposed to the primary message attachment, events that would occur as shown in noop are highlighted in
this color.  

The default is a lightly and airy sky blue, perhaps an azure dawn signalling your impending changes 
in this noop flagged event attachment.

### `String $slack_changed_event_color = '#EA950B'`
*GLOBAL OPTION ONLY IN THIS VERSION*

As opposed to the primary message attachment, event attachments that are associated with a `changed` status
of a run, are marked with this color.

The default is a ruinous mustard, a snort of pepper, or aged jaundice.

### `String $slack_failed_event_color = '#FF2D2D'`
*GLOBAL OPTION ONLY IN THIS VERSION*

As opposed to the primary message attachment, event attachments that are associated with a `failed` status
of a run, are marked with this color.

The default is a lightened red, blood and milk.

### `String $slack_unchanged_emoji = ':information_source:'`
*GLOBAL OPTION ONLY IN THIS VERSION*

Slack has emoji markup, where the name or alias in your workspace for the emoji 'this_emoji' is specified as
the name of the emoji surrounded by the ':' character. so e.g. `:slot_machine:`.

This emoji will appear as flair for at the top of your report messages with an uchanged status, 
in its text, and aids in visual highlighting of report status!

### `Integer[0, 98] $slack_max_attach_count = 98`

This integer value controls the maximum attachment count that can be put onto a reported message.
An attachment other than the main one present on all reports, and that present for included logs (if any),
are brought about and added when events occur within a run.  for example, a file changing ownership and
permissions would add two events.

This attachment behavior will occur by default, if `$slack_events_as_attach` is true. 

Otherwise, or if the number of events exceeds the max or absolute api cap, event data will be appended to
the text block of the primary attachment, so to prevent overflow.

But this doesn't (to the authors eye) look as nice on mobile clients.

However, on initial transform / conformance, when many resources might change, this behavior prevents a 
report from being lost, because it is too long.

There is potential for overflow of the text component in this coping mechanic, however. Again related to 
API enforcement criteria on the slack side.  This will result in message truncation when the event data text
is in excess of 100k characters.  They'll still show up, but will be marked as truncated, and will be.

Hopefully you do not have too many runs in which you'd include that much text -- or if you do, subsequent do
not show such radical continued flux in state.

### `Boolean $slack_events_as_attach = true`
*GLOBAL OPTION ONLY IN THIS VERSION*

Enabled by default, this controls report message construction - specifically with regard to how events
associated with a puppet run are assembled into a message sent to slack.  

The param `$slack_max_attach_count` (described in this section) explains how this works and why it is 
implemented a bit.

### `Boolean $slack_include_eval_time = false`
*GLOBAL OPTION ONLY IN THIS VERSION*

If set true, the evaluation time for a resource associated with a reported event will be included as a field.

Normally, you may find this detail extraneous, and overly verbose.  Thus is is 'off' / `false` by default.

### `Boolean $slack_include_run_time_metrics = false`
*GLOBAL OPTION ONLY IN THIS VERSION*

If set true, time metrics specified in the param `$slack_time_metrics_keys` will be appended as 'fields' to 
the main message attachment.

### `Optional[Array[String]] $slack_attach_log_tags = undef`
*GLOBAL OPTION ONLY IN THIS VERSION*

OPTIONAL - This parameter is an array of tags that events of ANY log level, should the event be tagged so,
will be included in the report log attachment sent to slack in a message.

For example:

~~~~
file { '/tmp/junkfile.txt':
  ensure  => present,
  owner   => 'puppet',
  group   => 'puppet',
  mode    => '0440',
  content => 'junker file - delete me if you dare, kupo.',
  tag     => 'junkytag',
}
~~~~

note the `tag` on this file resource in the example above.  were `$slack_attach_log_tags` array to include
the value 'junkytag' from our example, then ANY event associated with the tag will be appended to the slack
message in the logging attachment.

This can be extremely useful when you might wish to see info / notify log severity level events that are
associated with particular resources.

### `Optional[Array[String]] $slack_include_patterns = undef`
*GLOBAL OPTION -- OVERRIDES ANY SLACK ROUTING CONFIG*

If there is no value then there is no 'your self.host must match a pattern here or there is no slack send'.

If there is a regex match in `$slack_mute_patterns` array, it will override positive inclusion here.
This is by design. 

an array of string regex patterns, which will be used to match the 'self.host' of the report in question.
Evaluation is done early in the `process` method of `slack.rb`, prior to assembly of the message json sent.

Note, if this param is specified, then you have globally enabled report inclusion matching.  In other words,
if there is NOT a regex match for a nodes 'self.host' (which appears to be the puppet node cert name, hence 
typically fqdn) then there will be no slack reporting.

Why would you want positive only inclusion?  Consider an example; one may have two domains that are part 
of the fqdn we use for node cert names.  Perhaps we wish to only report on nodes that match regex for one
of those two domain suffices. 

(the format is like a regex string, but no surrounding /.../, it'll be exposed to `%r[#{s}]` where `s` is a 
string in the passed array in `slack.rb`, to be entirely explicit - I tried to make the format easy to use)

~~~
#<my hieradata module>/data/nodes/my_puppet_master.moogles.yaml
---
#...
report_slack::slack_include_patterns:
  - '^.*\.unclassified\.moogles*$'
~~~

### `Optional[Array[String]] $slack_mute_patterns = undef`
*GLOBAL OPTION -- OVERRIDES ANY SLACK ROUTING CONFIG*

if the 'self.host' in the report matches a regex pattern derived from strings in the passed array here 
(assuming you have created one) then a node will be excluded from reporting.

This will override any other routing or inclusion configuration - it does what it says, it mutes node(s)
by regex, and does so early in the `process` method of `slack.rb`.

this might be useful should you wish to mute particular nodes which are aberrant, and will never fix 
themselves without some kind of intervention, or silence a group of systems due to their sensitivity, or
other else.

### `Optional[Hash[String, Hash[String, Array[String]]]] $slack_routing_data = undef`

This is a hash of hashes, an example of which is given below, in which you can specify a key in the outer
hash of regex string to match, and with an optional (or empty, will just use config defaults for missing data)
inner hash containing the keys `report_states` and / or `webhooks` -- both of these are String Arrays.

Future versions of this module may include function for specifying other match specific options.

This feature may be useful in echoing to other channels in slack, those particular to a team using the node
as an example, or to echo failure state events to a global channel, or other else (you could target multiple)
slacks should you wish, its just a list of webhooks that should be fired if there's a regex match, and the 
status of the run matches either by consideration of the default or as you've specified.

it will assemble relevant entries, and while you may not repeat a key in the hash for regex match as is so,
if there are multiple relevant entries that apply to the current state, the list of relevant webhooks to 
engage will be appropriately constructed and de-duped in `slack.rb`.

~~~~
#<my hieradata module>/data/nodes/my_puppet_master.moogles.yaml
---
#...
report_slack::slack_default_webhook: 'https://hooks.slack.com/EXAMPLE/YOUR/WEB/HOOK'
report_slack::slack_default_statuses:
  - failed
  - changed
report_slack::slack_routing_data:
  '^secret-moogle.*$':
    report_states:
      - failed
    webhooks:
      - 'https://hooks.slack.com/services/this/that/other'
      - 'https://hooks.slack.com/services/some/web/hook'
      - 'https://hooks.slack.com/services/another/workspace/entirely'
  '^secret-moog.*$':
    report_states:
      - failed
      - changed
    webhooks:
      - 'https://hooks.slack.com/services/some/web/hook'
      - 'https://hooks.slack.com/services/iwont/be/excluded'
  '^secret-moo.*$': {}
  '^secret-mo.*$':
    report_states:
      - failed
      - changed
      - unchanged
~~~~

In the above example, we've overlapped regex patterns to match, such that the example node 
`secret-moogles.shinra.corp` will be considered against numerous matches.

going through them match by match (they all match in this case, lets pretend status is failed):

if the report status is *failed*:

`'^secret-moogle.*$'` - 

`['https://hooks.slack.com/services/this/that/other',
'https://hooks.slack.com/services/some/web/hook',
'https://hooks.slack.com/services/another/workspace/entirely']` 

are added to relevant hooks.

`'^secret-moog.*$'` - 

`['https://hooks.slack.com/services/some/web/hook',
'https://hooks.slack.com/services/iwont/be/excluded']`

are added to relevant hooks.

`'^secret-moo.*$'` - 

*there is an empty inner hash here, which means defaults are followed.  This is effectively adding default
values for state selection against current state, and uses default webhook.  It would be uncommon to use such
a configuration, as nodes with no relevant routing data revert to defaults.*

`'https://hooks.slack.com/EXAMPLE/YOUR/WEB/HOOK'`

is added to relevant hooks.

`'^secret-mo.*$'` - 

`'https://hooks.slack.com/EXAMPLE/YOUR/WEB/HOOK'`

is added to relevant hooks.


*this entirely overlaps, and specifies no particular webhook data, so because the example state is failed...
...the default webhook will be added to relevant hooks*

Finally, there is de-duplication for the webhooks to engage for this report processor run - meaning you won't
hit the same webhook more than once with the same report.

## Example Configuration using implicit lookup on params via hiera
A fuller configuration example, which would populate data for the class `report_slack` should you include it
in the catalog of a puppet master(s).

~~~~
#<my hieradata module>/data/nodes/my_puppet_master.moogles.yaml
---
---
report_slack::slack_default_webhook: 'https://hooks.slack.com/EXAMPLE/YOUR/WEB/HOOK'
report_slack::slack_include_eval_time: true
report_slack::slack_include_run_time_metrics: true
report_slack::slack_default_statuses:
  - failed
  - changed
report_slack::slack_attach_log_levels:
  - 'warning'
  - 'err'
  - 'alert'
  - 'emerg'
  - 'crit'
report_slack::slack_attach_log_tags:
  - 'junkytag'
report_slack::slack_time_metrics_keys:
  - 'catalog_application'
  - 'config_retrieval'
  - 'fact_generation'
  - 'exec'
  - 'total'
report_slack::slack_mute_patterns:
  - '^not-a-moogle.*$'

report_slack::slack_routing_data:
  '^secret-moogle.*$':
    report_states:
      - failed
    webhooks:
      - 'https://hooks.slack.com/services/this/that/other'
      - 'https://hooks.slack.com/services/some/web/hook'
      - 'https://hooks.slack.com/services/another/workspace/entirely'
  '^secret-moog.*$':
    report_states:
      - failed
      - changed
    webhooks:
      - 'https://hooks.slack.com/services/some/web/hook'
      - 'https://hooks.slack.com/services/iwont/be/excluded'
  '^secret-moo.*$': {}
  '^secret-mo.*$':
    report_states:
      - failed
      - changed
      - unchanged

classes:
  - 'report_slack'
~~~~

then in node classification, for example in `<control>/manifests/site.pp`:

~~~~
contain lookup('classes', Array[String], 'unique', [])
~~~~

## Temporarily Block the Report Processor Outwith Puppet
set env var `BLOCK_PUPPET_REPORT_SLACK` true.
