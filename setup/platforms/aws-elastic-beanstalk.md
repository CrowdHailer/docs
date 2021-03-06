---
description: Send AWS Elastic Bean Stalk logs to Timber
---

# AWS Elastic Bean Stalk

Timber integrates with [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/) through it's [ebextensions feature](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/ebextensions.html), enabling you to forward log files to to Timber.

## Installation

1. In your `shell` add the `TIMBER_API_KEY` and `TIMBER_SOURCE_ID` environment variables:  


   ```bash
   eb setenv TIMBER_API_KEY={{your-api-key}} TIMBER_SOURCE_ID={{your-source-id}}
   ```

   \(you can also do this in the [AWS EB console](https://console.aws.amazon.com/elasticbeanstalk) itself by following [these instructions](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environments-cfg-softwaresettings.html#environments-cfg-softwaresettings-console).\)

2. Add the [`01_timber.config` file](https://gist.github.com/binarylogic/26f97f4ef3589bdbdd14e65fd4c002a8) to your `.ebextensions` folder:  


   ```bash
   mkdir -p .ebextensions
   curl -o .ebextensions/01_timber_install.config https://gist.githubusercontent.com/binarylogic/26f97f4ef3589bdbdd14e65fd4c002a8/raw/ca9ab56fcc18c62d9e1ba22964e6e3e0c0f6915a/001_timber.config
   ```

3. Package your source bundle and upload it to AWS Elastic Beanstalk as usual.

## Configuration

The [`001_timber.config` file](https://gist.github.com/binarylogic/26f97f4ef3589bdbdd14e65fd4c002a8) installs [FluentBit](http://fluentbit.org/) as a log forwarder. As such, please refer to [FluentBit's configuration documentation](https://docs.fluentbit.io/manual/configuration) on the various options available. All of the options can be modified in the `"/tmp/td-agent-bit.conf":` section of the `001_timber.config` file.

## FAQs

### Which log files should I forward?

This depends on your use-case, but we've typically found the following log files to be valuable:

1. Apache HTTP Access Log - `/var/log/httpd/access_log`
2. Apache HTTP Error Log - `/var/log/httpd/error_log`
3. Nginx Access Log - `/var/log/nginx/access.log`
4. Nginx Error Log - `/var/log/nginx/error.log`
5. Nodejs - `/var/log/nodejs/nodejs.log`
6. Passenger - `/var/app/support/logs/passenger.log`
7. Rails - `/var/app/support/logs/production.log`

Please refer to the [AWS docs on Elastic Beanstalk log locations](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.logging.html#health-logs-instancelocation) for a comprehensive guide and list. If you are unsure, it is recommended that you SSH onto the server to locate the actual paths of your log files.

## Troubleshooting

Because the AWS Elastic Bean Stalk integration relies on [Fluent Bit](../log-forwarders/fluent-bit.md) we recommend reviewing the following troubleshooting guides:

{% page-ref page="../../guides/troubleshooting-log-delivery.md" %}

{% page-ref page="../log-forwarders/fluent-bit.md" %}

