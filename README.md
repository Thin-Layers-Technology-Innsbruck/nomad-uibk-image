<img src="assets/nomad_oasis_logo.png" alt="NOMAD Oasis Logo" width="200">

# NOMAD Oasis Distribution

![docker image](https://github.com/Thin-Layers-Technology-Innsbruck/nomad-UIBK-image/actions/workflows/docker-publish.yml/badge.svg)

This is an distribution image of a [NOMAD Oasis](https://nomad-lab.eu/nomad-lab/nomad-oasis.html) provided by [FAIRmat](https://github.com/FAIRmat-NFDI).
Below are instructions on how to [deploy this distribution](#deploying-the-distribution)
and how to customize it by [adding plugins](#adding-a-plugin).

## Deploying the distribution

Below are instructions on how to deploy this NOMAD Oasis distribution either
[for a new Oasis](#for-a-new-oasis) or [for an existing Oasis](#for-an-existing-oasis).
For further questions, consult the [documentation](https://nomad-lab.eu/prod/v1/docs/oasis/install.html).

### For a new Oasis

1. Make sure you have [docker](https://docs.docker.com/engine/install/) installed.
Docker nowadays comes with `docker compose` built in. Prior, you needed to
install the stand-alone [docker-compose](https://docs.docker.com/compose/install/).

2. Clone the repository or download the repository as a zip file.

```sh
git clone https://github.com/Thin-Layers-Technology-Innsbruck/nomad-UNITOV-image.git
cd nomad-UNITOV-image
```

or

```sh
curl-L -o nomad-UNITOV-image.zip "https://github.com/Thin-Layers-Technology-Innsbruck/nomad-UNITOV-image/archive/main.zip"
unzip nomad-UNITOV-image.zip
cd nomad-UNITOV-image
```

3. _On Linux only,_ recursively change the owner of the `.volumes` directory to the nomad user (1000) 

```sh
sudo chown -R 1000 .volumes
```

4. Pull the images specified in the `docker-compose.yaml`.

```sh
docker compose pull
```

5. And run it with docker compose in detached (--detach or -d) mode 

```sh
docker compose up -d
```

6. Optionally you can now test that NOMAD is running with

```
curl localhost/nomad-oasis/alive
```

8. Finally, open [http://localhost/nomad-oasis](http://localhost/nomad-oasis) in your browser to start using your new NOMAD Oasis.

#### Updating the Oasis

Whenever you want to update your image you first need to shut down NOMAD using `docker compose down`. Afterwards you can pull the updates and simply restart the oasis:
```sh
docker compose down
docker compose pull
docker compose up -d
```

#### NOMAD Remote Tools Hub (NORTH)

To run NORTH (the NOMAD Remote Tools Hub), the `hub` container needs to run docker and
the container has to be run under the docker group. You need to replace the default group
id `991` in the `docker-compose.yaml`'s `hub` section with your systems docker group id.
Run `id` if you are a docker user, or `getent group | grep docker` to find your
systems docker gid. The user id 1000 is used as the nomad user inside all containers.

You can find more details on setting up and maintaining an Oasis in the NOMAD docs here:
[nomad-lab.eu/prod/v1/docs/oasis/install.html](https://nomad-lab.eu/prod/v1/docs/oasis/install.html)

### For an existing Oasis

If you already have an Oasis running you only need to change the image being pulled in
your `docker-compose.yaml` with `ghcr.io/Thin-Layers-Technology-Innsbruck/nomad-UIBK-image:main` for the services
`worker`, `app`, `north`, and `logtransfer`.

If you want to use the `nomad.yaml` from this repository you also need to comment out
the inclusion of the `nomad.yaml` under the volumes key of those services in the
`docker-compose.yaml`.

```yaml
    volumes:
      # - ./configs/nomad.yaml:/app/nomad.yaml
```

To run the new image you can follow steps 5. and 6. [above](#for-a-new-oasis).

## Adding a plugin

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

Once the changes have been committed to the main branch, the new image will automatically
be generated.

## The Jupyter image

In addition to the Docker image for running the oasis, this repository also builds a custom NORTH image for running a jupyter hub with the installed plugins.
This image has been added to the [`configs/nomad.yaml`](configs/nomad.yaml) during the initialization of this repository and should therefore already be available in your Oasis under "Analyze / NOMAD Remote Tools Hub / jupyter"

The image is quite large and might cause a timeout the first time it is run. In order to avoid this you can pre pull the image with:

```
docker pull ghcr.io/fairmat-nfdi/nomad-distribution-template/jupyter:main
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


## Updating the distribution from the template

In order to update an existing distribution with any potential changes in the template you can add a new `git remote` for the template and merge with that one while allowing for unrelated histories:

```
git remote add template https://github.com/FAIRmat-NFDI/nomad-distribution-template
git fetch template
git merge template/main --allow-unrelated-histories
```

Most likely this will result in some merge conflicts which will need to be resolved. At the very least the `Dockerfile` and GitHub workflows should be taken from "theirs":

```
git checkout --theirs Dockerfile
git checkout --theirs .github/workflows/docker-publish.yml
```

For detailed instructions on how to resolve the merge conflicts between different version we refer you to the latest [template release notes](https://github.com/FAIRmat-NFDI/nomad-distribution-template/releases/latest).

Once the merge conflicts are resolved you should add the changes and commit them

```
git add -A
git commit -m "Updated to new distribution version"
```

Ideally all workflows should be triggered automatically but you might need to run the initialization one manually by navigating to the "Actions" tab at the top, clicking "Template Repository Initialization" on the left side, and triggering it by clicking "Run workflow" under the "Run workflow" button on the right.


## FAQ/Trouble shooting

 *I get an* `Error response from daemon: Head "https://ghcr.io/v2/Thin-Layers-Technology-Innsbruck/nomad-UIBK-image/manifests/main": unauthorized`
 *when trying to pull my docker image.*
 
 Most likely you have not made the package public or provided a personal access token (PAT).
 You can read how to make your package public in the GitHub docs [here](https://docs.github.com/en/packages/learn-github-packages/configuring-a-packages-access-control-and-visibility)
 or how to configure a PAT (if you want to keep the distribution private) in the GitHub
 docs [here](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-with-a-personal-access-token-classic).

## Acknowledgments

Funding for this work has been provided by the European Union as part of the SolMates project (Project Nr. 101122288).

<img src="assets/eu_funding_logo.png" alt="EU Funding Logo" width="300">
<img src="assets/solmates_logo.png" alt="SolMates Logo" width="300">
