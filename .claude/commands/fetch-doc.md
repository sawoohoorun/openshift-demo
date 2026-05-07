# fetch-doc

Fetch and summarize OpenShift or Kubernetes documentation for a topic.

Usage: /fetch-doc <topic>

Steps:
1. Use WebSearch to find the most relevant official docs page for the topic
2. Use WebFetch to retrieve the page content
3. Summarize the key points relevant to the current project context
4. Include the source URL

Example: /fetch-doc "OpenShift image streams"
