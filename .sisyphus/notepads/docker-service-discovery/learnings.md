# Docker Service Discovery Learnings

## Documentation Structure

The docs/docker-labels.md file follows a clear structure:
1. Quick Example at the top for immediate understanding
2. How It Works explanation
3. CLI Flags reference
4. Docker Socket Configuration
5. Label Reference (broken down by container type)
6. Advanced Features
7. Complete Examples
8. Docker Networking
9. Troubleshooting

## New Feature Documentation Pattern

Added new section "### Service Definitions on Tsbridge Container" after "### On Service Containers" section. This maintains the logical flow:
- First: labels on tsbridge container (global config)
- Second: labels on service containers (traditional discovery)
- Third: service definitions on tsbridge container (new discovery mechanism)

## Key Documentation Elements Included

1. **Clear use case explanation**: Dynamic containers like Nextcloud AIO
2. **Label format documentation**: `tsbridge.service.{name}.{property}`
3. **Practical YAML example**: Shows tsbridge container with service labels
4. **Comparison table**: Distinguishes from `tsbridge.enabled=true` approach
5. **Coexistence explanation**: Both mechanisms work together, traditional takes precedence
6. **Real-world example**: Nextcloud AIO with external network

## Style Consistency

- Used same code block formatting as existing docs
- Used same heading levels (### for sections, #### for subsections)
- Used same table format for comparisons
- Used same note/callout style (plain text, no emojis)
- Maintained same level of detail and explanation depth

## Implementation Understanding

From internal/docker/labels.go (parseServiceNames):
- Parses labels matching pattern `{prefix}.service.{name}.{property}`
- Extracts unique service names from matching labels
- Returns slice of service names for container lookup

From internal/docker/docker.go (Load function):
- First finds tsbridge container for global config
- Parses service names from tsbridge container labels
- Finds containers by those names
- Also finds containers with `tsbridge.enabled=true`
- Merges both discovery mechanisms
- Traditional discovery (`tsbridge.enabled=true`) takes precedence
- Docker provider allows zero services at startup

## Label Format

Format: `{prefix}.service.{name}.{property}`
Example: `tsbridge.service.nextcloud.port=80`

Where:
- `{prefix}`: Configurable label prefix (default: "tsbridge")
- `{name}`: Service name (matches container name)
- `{property}`: Service property (port, tags, etc.)
