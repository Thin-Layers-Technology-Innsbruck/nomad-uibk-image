## To do

### Configuring Secure HTTP and HTTPS Connections

   By default `docker-compose.yaml` uses the HTTP protocol for communication. This works for testing, but before entering production you must secure your setup with HTTPS; otherwise, any communication with the server—including credentials and sensitive data—can be compromised.

   HTTPS requires a TLS certificate, which must be renewed periodically. Depending on your setup, you have several options:

   1. You already have a certificate.

      In this case, you just need the certificate and key files.

   2. Free certificate from Let's Encrypt

      [Let's Encrypt](https://letsencrypt.org/) provides free TLS certificates for those with a domain name. Follow their tutorials for instructions on generating a certificate.

   3. Self-signed certificate

      For testing, you can create a [self-signed certificate](https://en.wikipedia.org/wiki/Self-signed_certificate). Note that self-signed certificates are not recommended for production since they are not trusted by browsers. You can generate one with:

      ```sh
      mkdir ssl
      openssl req -x509 -nodes -days 365 \
        -newkey rsa:2048 \
        -keyout ./ssl/selfsigned.key \
        -out ./ssl/selfsigned.crt \
        -subj "/CN=localhost"
      ```

   To start using a TLS certificate, update the `proxy` configuration in `docker-compose.yml`:
   ```diff
   - # HTTP
   - - ./configs/nginx_http.conf:/etc/nginx/conf.d/default.conf:ro

   + # HTTPS
   + - ./configs/nginx_https.conf:/etc/nginx/conf.d/default.conf:ro
   + - ./ssl:/etc/nginx/ssl:ro  # Your certificate files
   ```

## How to
### Updating the image
Any pushes to the main branch of this repository, such as when [adding a plugin](#adding-a-plugin), will trigger a pipeline that generates a new app and jupyter image.

1. To update your local image you need to shut down NOMAD using

    ```sh
    docker compose down
    ```
    pull the newly created image
    ```sh
    docker compose pull
    ```
    and bring the Oasis back up with
    ```sh
    docker compose up
    ```

2. You can remove unused images to free up space by running

    ```sh
    docker image prune -a
    ```

### Adding a plugin

To add a new plugin to the docker image you should add it to the plugins table in the [`pyproject.toml`](pyproject.toml) file.

Here you can put either plugins distributed to PyPI, e.g.

```toml
[project.optional-dependencies]
plugins = [
  "nomad-material-processing>=1.0.0",
]
```

or plugins in a git repository with either the commit hash

```toml
[project.optional-dependencies]
plugins = [
  "nomad-measurements @ git+https://github.com/FAIRmat-NFDI/nomad-measurements.git@71b7e8c9bb376ce9e8610aba9a20be0b5bce6775",
]
```

or with a tag

```toml
[project.optional-dependencies]
plugins = [
  "nomad-measurements @ git+https://github.com/FAIRmat-NFDI/nomad-measurements.git@v0.0.4"
]
```

To add a plugin in a subdirectory of a git repository you can use the `subdirectory` option, e.g.

```toml
[project.optional-dependencies]
plugins = [
  "ikz_pld_plugin @ git+https://github.com/FAIRmat-NFDI/AreaA-data_modeling_and_schemas.git@30fc90843428d1b36a1d222874803abae8b1cb42#subdirectory=PVD/PLD/jeremy_ikz/ikz_pld_plugin"
]
```

Once the changes have been committed to the main branch, the new image will automatically be generated.

### The Jupyter image

In addition to the Docker image for running the oasis, this repository also builds a custom NORTH image for running a jupyter hub with the installed plugins.
This image has been added to the [`configs/nomad.yaml`](configs/nomad.yaml) during the initialization of this repository and should therefore already be available in your Oasis under "Analyze / NOMAD Remote Tools Hub / jupyter"

We currently use `quay.io/jupyter/base-notebook:2025-04-14` as our base image for Jupyter. While it includes the necessary Python packages, it does not come with `R` or `Julia` pre-installed.
If you need support for those languages, you can switch to `quay.io/jupyter/datascience-notebook:2025-04-04`, which includes both `R` and `Julia`.
The Jupyter image does not include `gcc` or `build-essential` by default. If you want to allow users to install Python packages that require compilation while running a notebook, you'll need to install these tools in the [Dockerfile](./Dockerfile#L172) or switch the base image to `quay.io/jupyter/datascience-notebook:2025-04-04`.
However, including these packages can increase the image size and may introduce security risks if arbitrary code is compiled at runtime.

Note that the `base-notebook` image is more lightweight and uses less disk space compared to the `datascience-notebook` image.

The image is quite large and might cause a timeout the first time it is run. In order to avoid this you can pre pull the image with:

```sh
docker pull ghcr.io/thin-layers-technology-innsbruck/nomad-uibk-image/jupyter:main
```

If you want additional python packages to be available to all users in the jupyter hub you can add those to the jupyter table in the [`pyproject.toml`](pyproject.toml):

```toml
[project.optional-dependencies]
jupyter = [
  "voila",
  "ipyaggrid",
  "ipysheet",
  "ipydatagrid",
  "jupyter-flex",
]
```

### Automated Unit and Example Upload Tests in CI

By default, all unit tests from every plugin are executed to ensure system stability and catch potential issues early. These tests validate core functionality and help maintain consistency across different plugins.

In addition to unit tests, the pipeline also verifies that all example uploads can be processed correctly. This ensures that any generated entries do not contain error messages, providing confidence that data flows through the system as expected.

For example upload tests, the CI uses the image built in the Build Image step. It then runs the Docker container and starts up the application to confirm that it functions correctly. This approach ensures that if the pipeline passes, the app is more likely to run smoothly in a Dockerized environment on a server, not just locally.

If you need to disable tests for specific plugins, update the **PLUGIN_TESTS_PLUGINS_TO_SKIP** variable in [.github/workflows/docker-publish.yml](./.github/workflows/docker-publish.yml#L21) by adding the plugin names to the existing list.

### Set Up Regular Package Updates with Dependabot

> This has been done, left for documentation

Dependabot is already configured in the repository’s CI setup, but you need to enable it manually in the repository settings.

To enable Dependabot, go to Settings > Code security and analysis in your GitHub repository. From there, turn on Dependabot alerts and version updates. Once enabled, Dependabot will automatically check for dependency updates and create pull requests when new versions are available.

This automated process helps ensure that your dependencies stay up to date, improving security and reducing the risk of vulnerabilities.

### Customizing Documentation

By default, documentation is built using the [nomad-docs](https://github.com/Thin-Layers-Technology-Innsbruck/nomad-docs) repository. However, if you'd like to customize the documentation for your Oasis instance, you can easily do so.

1. First, [fork the nomad-docs repository](https://github.com/Thin-Layers-Technology-Innsbruck/nomad-docs/fork).
2. Make your desired changes in your fork.
3. Update the `NOMAD_DOCS_REPO` variable in the [.github/workflows/docker-publish.yml](./.github/workflows/docker-publish.yml#L19) file to point to the URL of your forked repository.

This setup ensures that your custom documentation is used when building your Oasis.


### Updating the distribution from the template

In order to update an existing distribution with any potential changes in the template you can add a new `git remote` for the template and merge with that one while allowing for unrelated histories:

```sh
git remote add template https://github.com/FAIRmat-NFDI/nomad-distro-template
git fetch template
git merge template/main --allow-unrelated-histories
```

Most likely this will result in some merge conflicts which will need to be resolved. At the very least the `Dockerfile` and GitHub workflows should be taken from "theirs":

```sh
git checkout --theirs Dockerfile
git checkout --theirs .github/workflows/docker-publish.yml
```

The lock file merge conflicts can be resolved to use your versions instead of the template repository resolution.
```sh
git checkout --ours uv.lock
```


For detailed instructions on how to resolve the merge conflicts between different version we refer you to the latest template release [notes](https://github.com/FAIRmat-NFDI/nomad-distro-template/releases/latest)

Once the merge conflicts are resolved you should add the changes and commit them

```sh
git add -A
git commit -m "Updated to new distribution version"
```

Ideally all workflows should be triggered automatically but you might need to run the initialization one manually by navigating to the "Actions" tab at the top, clicking "Template Repository Initialization" on the left side, and triggering it by clicking "Run workflow" under the "Run workflow" button on the right.
