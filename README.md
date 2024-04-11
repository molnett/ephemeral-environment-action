# Molnett Ephemeral Environments

This Github action makes it easier to use the ephemeral environments feature of molnett. The feature is intended for creating a deployment for each pull request.

## Usage

### Minimal example

Pull request build:

```yaml
      - name: Deploy Ephemeral Environment
        id: deploy
        uses: molnett/ephemeral-environment-action@v1
        with:
          action: deploy
          manifest-path: molnett.yaml
```

Main branch build:

```yaml
    - name: Delete Ephemeral Environment
      id: delete
      uses: molnett/ephemeral-environment-action@v1
      with:
        action: delete
        manifest-path: molnett.yaml
```

### Full example with setup, docker build and pr comment

```yaml
name: "PR"

on:
  pull_request:
    types:
      - opened
      - edited
      - synchronize

jobs:
  create-ephemeral-env:
    name: Ephemeral Environment
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - name: Molnctl Setup
        uses: molnett/setup-molnctl-action@v2
        with:
          api-token-client-id: ${{ secrets.MOLNETT_CLIENT_ID }}
          api-token-client-secret: ${{ secrets.MOLNETT_CLIENT_SECRET }}
          default-org: demo
      - name: Build & Push Image to Molnett
        run: |
          molnctl auth docker
          IMAGE_NAME=`molnctl svcs image-name -u molnett.yaml`
          docker buildx build . -t $IMAGE_NAME
          docker push $IMAGE_NAME
      - name: Deploy Ephemeral Environment
        id: deploy
        uses: molnett/ephemeral-environment-action@v1
        with:
          action: deploy
          manifest-path: molnett.yaml
      - name: Comment Domain
        uses: actions/github-script@v7
        with:
          script: |
            const {data: comments} = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.payload.number,
            })
            const botComment = comments.find(comment => comment.user.id === 41898282)

            if (!botComment) {
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: `👋 Your service *${{ steps.deploy.outputs.service-name }}* in the environment *${{ steps.deploy.outputs.environment-name }}* has been deployed at https://${{ steps.deploy.outputs.environment-name }}-${{ steps.deploy.outputs.service-name }}.mltt.io!`
              })
            } else {
              github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
                body: `👋 Your service *${{ steps.deploy.outputs.service-name }}* in the environment *${{ steps.deploy.outputs.environment-name }}* has been deployed at https://${{ steps.deploy.outputs.environment-name }}-${{ steps.deploy.outputs.service-name }}.mltt.io!`
              })
            }
      - name: Cleanup
        run: rm -r ~/.config/molnett
```

## OS support

The action only supports Linux for now.
