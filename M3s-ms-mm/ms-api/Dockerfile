# Stage 1: Build
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Install dependencies
RUN apk add --no-cache git

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -o ms-api .

# Stage 2: Run
FROM alpine:3.19

WORKDIR /app

# Install ca-certificates for HTTPS calls
RUN apk add --no-cache ca-certificates tzdata

# Copy binary from builder
COPY --from=builder /app/ms-api .

# Expose port
EXPOSE 8002

# Run
CMD ["./ms-api"]
