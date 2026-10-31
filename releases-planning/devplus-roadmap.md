---
id: 30af092d-be76-4943-88b6-fd7f85ad3e4b
binder: binder-mvvle32a-1
folder: folder-mvw0u899
---
DevPlus Roadmap
## Done 
Features (ideas)
- Since we now have discovery on GitHub Repos, and we seem to be able to categorize what is an eligible terraform module. 
- We need environments in DevPlus (Dev, Stage, Prod)... or do we think Projects can be Environments ? 
- What does this project cost per month, and what's driving the spend?
- Should we surface those as candidates for registry import?
- Since we also have index on repos that call a module Should we also have usage count on terraform modules 
- drift detection seems to be failing on every adopted resource. 
- Feature to allow Persist a module that exists only in DevPlus to Source Control. 
- Scheduled drift detection runs as well as module refreshes frequency should be configurable on the platform, with sensible limits.
- I want health to move beyond drift detection only. This will be a big feature and it will rely exclusively on what each connector provider offers and the resource type. The idea here is once we provision a resource, it becomes our responsibility to ensure its health. As an example, in AWS cloud watch alarms + cloud trail events should have some data 
- Agent that writes architectural principles.
  - Agent that writes cedar policies.
- I should be able to see whats driving up cost in my account
- A way to securely pass things like username/pw to a user if its generated in state.
- Rip out InfraCost, build our own. 
## *In Progress*

- Can we view a more ux sexy view of the resources being created Like icons loading bars and such (From parsing the logs), logs should still be viewable if we choose. 

- Live Trace Viewer for AI interactions.
  - Where are they stored?
  - What is our retention on these ? 
  - **Streamed to SIEM (Splunk, etc) * *
  - Drift detection should show what properties would change on a resource. 

- Previews should expire after some time (24 hours?)
- Previews are counting towards the total cost of the project (They should not be, nothing was actually deployed)

## To Do 
- Where can I set #Tiers? 
- I dont see the ability to #promote a deployment in the UI. 

## Hygiene
- Look into the connection pooling issue saamer brought up in his PR 

Release (Only when we are done with the features for this release)
- Look at every open PR and see if there is value in there, if so bundle with the release prs, and close the stale ones.
