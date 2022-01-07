# Summary # 

Cloud Platform is a glue that connects several 3rd party services (Azure SQL, Azure Storage Account, Azure Kubernetes Service, Better Uptime, Seq, SumoLogic, etc.). End to end (E2E) tests have been so far the most reliable way to ensure the glue works. This is why Cloud Portal heavily relies on them.


# Details #

Cloud Portal uses different types of tests (unit, integration, end to end (E2E))) to make sure the code we ship is rock solid. That being said, we rely on E2E Tests much more than what one would find in other apps.  

Cloud Portal has only a few places wih complex logic. The rest of the code makes sure that Cloud Platform infrastructure is configured in the way we want it to be configured.

We could mock 3rd party API calls and assume that everything is fine. Weve learnt that the 3rd party services are build by people like us and are from from being perfect. APIs change without a notice, APIs runtime characterstices change (e.g. a race condition is itnroduced). Our E2E Tests are part of the QA deparment. Technically speaking we should need to do that but telling our cusotmers that can't deploy because a 3rd part sergvice introduced a bug or did not honour contract is not a good answer.

