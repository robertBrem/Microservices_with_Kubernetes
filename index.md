# Microservices with Kubernetes

* Setup
  * [Setup your local host machine](01_Setup/01_Host_setup.md)
  * [Setup a Kubernetes cluster](01_Setup/02_Kubernetes_setup.md)
* First REST service
  * [Creating our first JavaEE microservice](02_First_rest_service/01_JavaEE_service.md)
  * [Dockerization of the service](02_First_rest_service/02_Dockerization.md)
  * [Deploy the Docker image](02_First_rest_service/03_Deploy_in_Kubernetes.md)
* [Docker registry](03_Docker_registry/01_Setup_a_docker_registry.md)
* [Jenkins](04_Jenkins/01_Setup_Jenkins_in_Kubernetes.md)
* [Build and push](05_Build_and_push_ci_step/01_Build_and_push_CI_step.md)
* Systemtest
  * [Setup the test environment](06_Systemtest_ci_step/01_Setup_test_env.md)
  * [Add a system test](06_Systemtest_ci_step/02_Systemtest.md)
  * [Start the Jenkins pipeline on every push](06_Systemtest_ci_step/03_Start_pipeline_on_every_push.md)
* [UI test](07_UI_test/01_UI_Test.md)
* [Lasttest](08_Lasttest/01_Last_test.md)
* [Manual test](09_Manual_test/01_Create_manual_test.md)
* [Canary release](10_Canary_release/01_Create_a_canary_release.md)
* [Full production release](11_Full_production_release/01_Go_full_production.md)
* [Own Git repository](12_Own_Gogs/01_Use_your_own_Git_repository.md)
* Monitoring
  * [Kubernetes dashboard](13_Monitoring/01_Dashboard.md)
  * [Monitoring with Prometheus](13_Monitoring/02_Prometheus.md)
  * [Weave Scope](13_Monitoring/03_Weave_scope.md)
* [Documentation of a REST service](14_Documentation_of_rest_service/01_Documentation.md)
* Angular2 frontend
  * [Create an Angular 2 frontend](15_Angular2_frontend/01_Initial_setup.md)
  * [Create a test and production stage](15_Angular2_frontend/02_Test_and_prod_stage.md)
  * [Creation of the first view](15_Angular2_frontend/03_First_view.md)
* SSO with Keycloak
  * [Setup Keycloak](16_SSO_with_Keycloak/01_Setup_Keycloak.md)
  * [Initial setup of Keycloak](16_SSO_with_Keycloak/02_Initial_setup_of_Keycloak.md)
  * [Secure the `jax-rs` service](16_SSO_with_Keycloak/03_Secure_REST_service.md)
  * [Secure the Angular2 frontend service](16_SSO_with_Keycloak/04_Secure_frontend_service.md)
* Implement Event Sourcing with Cassandra
  * [Setup Cassandra](17_Event_Sourcing_with_Cassandra/01_Setup_Cassandra.md)
  * [Change the REST service to use event sourcing](17_Event_Sourcing_with_Cassandra/02_REST_service_with_event_sourcing.md)
  * [Change to the frontend service](17_Event_Sourcing_with_Cassandra/03_Changes_to_the_frontend.md)
* Connect Keycloak user with application user
  * [Setup Keycloak with an event provider](18_Connect_keycloak_user_with_application_user/01_Setup_keycloak_with_a_provider.md)
  * [Changes we've to do in the REST service](18_Connect_keycloak_user_with_application_user/02_Changes_to_the_REST_service.md)
  * [Changes in the environment scripts](18_Connect_keycloak_user_with_application_user/03_Changes_in_the_environment.md)
  