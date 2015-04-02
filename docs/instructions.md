Using bosh-lite; apply 5GB file upload patch - https://groups.google.com/a/cloudfoundry.org/forum/#!msg/bosh-users/MjiFAdpyimQ/VeOCpG9SsHQJ

Then: 

Upload cf-release

```
bosh upload release https://bosh.io/d/github.com/cloudfoundry/cf-release?v=205
```

Upload logsearch-boshrelease

```
bosh upload release https://logsearch-boshrelease.s3.amazonaws.com/boshrelease-logsearch-18%2Bdev.64.tgz
```

Upload logsearch-for-cloudfoundry-boshrelease

```
bosh upload release https://logsearch-for-cloudfoundry-boshrelease.s3.amazonaws.com/boshrelease-logsearch-for-cloudfoundry-0%2Bdev.3.tgz
```

Make 3 deployments using the manifests below:

[cf-warden.yml](./cf-warden.yml)

[logsearch-for-cloudfoundry-warden.yml](./logsearch-for-cloudfoundry-warden.yml)

[vagrant-logsearch.yml](./vagrant-logsearch.yml)
