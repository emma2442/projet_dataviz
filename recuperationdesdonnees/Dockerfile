# Use Alpine for a minimal base image
FROM alpine:latest

# Install bash, curl and jq
RUN apk add --no-cache bash curl jq

# Copy the script into the container
COPY recup.sh /recup.sh

# Ensure the script is executable
RUN chmod +x /recup.sh

# Command to run the script
CMD ["/recup.sh"]
