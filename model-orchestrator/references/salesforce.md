# Salesforce layer

Load only when the task is Salesforce work.

- **Build order:** objects and fields → Apex and flows → LWC → tests → permissions.
- **One deployment per org at a time:** batch deploys into one step, never from parallel agents.
- **Validate before deploying:** run a validate-only deploy plus tests. Failed tests count as a failed chunk.
- **Apex review:** check for bulkification and governor limits. Haiku should not write Apex classes.
- **Org access:** before relying on it, confirm the sub-agents actually have the Salesforce tools or connector in this session.
- **Production:** deploying to production needs the user's explicit yes (principle 8).
