===================================================
 FIHealth Sanity Checks | Publishing Sanity Status
===================================================

Results included in summary report ``test_results.txt`` are published to a
`Context Broker`_ (therefore, available for querying and stored in database).
Even more, we take advantage of the `subscription capabilities`__ of the
broker, so that results are notified to subscribers, being
`FIHealth Dashboard </dashboard/README.rst>`_ one of them.

__ `Context Broker - Context subscriptions`_

To do that, a request to the `NGSI Adapter`_ component must be issued. The name
of the probe originating source data will be ``sanity_tests``::

    $ curl "{adapter_endpoint}/sanity_tests?id={myregion}&type=region" \
    --header "Content-Type: text/plain" --header "txId: $BUILD_NUMBER" \
    -X POST -s -S -d @test_results.txt

Please note that the invocation to NGSI Adapter, once summary report has been
generated, is **only** automated when running the Sanity Checks within Jenkins.
Job's build number is used as transaction identifier for traceability.

As a prerequisite, custom parser ``resources/parsers/sanity_tests.js`` (provided
as part of this component) has to be installed together with the rest of parsers
bundled in NGSI Adapter package.

Such parser will extract NGSI attributes from the summary report and then invoke
Context Broker to update region context. In this process the **Sanity Status**
is calculated depending the results of individual tests.

Here is an example of the generated ``test_results.txt`` for an specific region
(*my_region*) is the following:

::

    [Tests: 28, Errors: 0, Failures: 0, Skipped: 0] | BUILD_NUMBER=12345

    REGION GLOBAL STATUS

    Regions satisfying all key test cases: ['test_(.*)']
     NONE!!!!!!!

    Regions only failing in optional test cases: ['test_.*container.*']
     >> my_region

    *********************************

    REGION TEST SUMMARY REPORT:

     >> my_region
      OK	 test_allocate_ip
      OK	 test_base_image_for_testing_exists
      OK	 test_cloud_init_aware_images
      NOK	 test_create_big_object_and_download_it_from_container
      OK	 test_create_container
      OK	 test_create_keypair
      OK	 test_create_network_and_subnet
      OK	 test_create_router_external_network
      OK	 test_create_router_no_external_network
      OK	 test_create_router_no_external_network_and_add_network_port
      OK	 test_create_security_group_and_rules
      N/A	 test_create_text_object_and_download_it_from_container
      OK	 test_delete_an_object_from_a_container
      OK	 test_delete_container
      OK	 test_deploy_instance_with_new_network
      OK	 test_deploy_instance_with_new_network_and_all_params
      OK	 test_deploy_instance_with_new_network_and_associate_public_ip
      OK	 test_deploy_instance_with_new_network_and_check_metadata_service
      OK	 test_deploy_instance_with_new_network_and_e2e_connection_using_public_ip
      OK	 test_deploy_instance_with_new_network_and_e2e_snat_connection
      OK	 test_deploy_instance_with_new_network_and_keypair
      OK	 test_deploy_instance_with_new_network_and_metadata
      OK	 test_deploy_instance_with_new_network_and_sec_group
      OK	 test_deploy_instance_with_shared_network_and_e2e_connection_using_public_ip
      OK	 test_deploy_instance_with_shared_network_and_e2e_snat_connection
      OK	 test_external_networks
      OK	 test_flavors_not_empty
      OK	 test_images_not_empty


Sanity Check Status and Test Execution Status
---------------------------------------------

Results of tests execution are written to a xUnit file ``test_results.xml``
(basename may be changed using ``--output-name`` command line option), and
additionally an HTML report ``test_results.html`` (or the same basename as
the former) is generated from the given template (or the default found at
``resources/templates/`` folder).

The script ``commons/result_analyzer.py`` is invoked to create a summary
report ``test_results.txt``. It will analyze the status of each region using
the *key_test_cases* and *opt_test_cases* information configured in the
``etc/settings.json`` file: a region is considered **OK** if all its test
cases with names matching the regular expressions defined in the first property
have been *PASSED*. If some of these *key test cases* are failed but these ones
are defined in the second property, the status of the region will
be **Partial OK**.

The aim of the *Sanity Check Status* is to be used as the value for the
**sanity_check_status** context attribute to be sent to Context Broker
by NGSI Adapter. Additional context attributes are sent to Context Broker,
one for each individual sanity test (named **sanity_check_***).

Therefore, we have two type of statuses:

- **Sanity Check Status**. The responsible for calculating this value is the
  custom parser for the NGSI Adapter aforementioned, whose input is the summary
  report ``test_results.txt`` that has been shown:

  +---------------+---------------------------------------------+
  | Sanity Status | Description                                 |
  +===============+=============================================+
  | *OK*          | All test cases defined in *key_test_cases*  |
  |               | have been **PASSED**                        |
  +---------------+---------------------------------------------+
  | *POK*         | Some of the test cases defined in           |
  |               | *key_test_cases* have been                  |
  |               | **FAILED/SKIPPED** but these ones are       |
  |               | defined in *opt_test_cases* config property |
  +---------------+---------------------------------------------+
  | *NOK*         | Some of the test cases defined in           |
  |               | *key_test_cases* have been                  |
  |               | **FAILED/SKIPPED** but these ones are NOT   |
  |               | defined in *opt_test_cases* config property |
  +---------------+---------------------------------------------+


- **Test status**. The ``commons/result_analyzer.py`` script generates the
  ``test_results.txt`` report with the list of executed test cases and their
  results, and also 'marks' the region considering the *key_test_cases*
  and *opt_test_cases*. Result of tests could be:

  +-------------+---------------------------------------------+
  | Test Status | Description                                 |
  +=============+=============================================+
  | OK          | The test has passed OK                      |
  +-------------+---------------------------------------------+
  | N/A         | The test should be executed but             |
  |             | preconditions to run that one               |
  |             | are not accomplished                        |
  +-------------+---------------------------------------------+
  | NOK         | The test has been executed but some         |
  |             | assertions has failed or an exception       |
  |             | has been raised                             |
  +-------------+---------------------------------------------+


Timestamp and elapsed execution time of Sanity Checks
-----------------------------------------------------

Apart from the context attributes described above, when Sanity Checks
have been executed from Jenkins a new attribute ``sanity_check_elapsed_time``
is updated in the context information of the region: the elapsed time of the
execution in milliseconds. And the parser in NGSI Adapter will also add
the ``sanity_check_timestamp`` attribute.


.. REFERENCES

.. _NGSI Adapter: https://github.com/telefonicaid/fiware-monitoring/tree/master/ngsi_adapter
.. _Context Broker: http://github.com/telefonicaid/fiware-orion/tree/master
.. _Context Broker - Context subscriptions: https://github.com/telefonicaid/fiware-orion/blob/master/doc/manuals/user/walkthrough_apiv1.md#context-subscriptions
