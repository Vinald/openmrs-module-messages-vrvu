# openmrs-module-messages — Connect for Life / OpenMRS

## What this repo is

- **Module**: `openmrs-module-messages` (Maven artifact id: `messages`, module id: `messages`)
- **Purpose**: Receives and schedules messages (Call or SMS) between a health facility and patients/caregivers —
  adherence reporting (pill reminders), adherence feedback, visit reminders, health tips, and surveys. Health
  facility staff can schedule messages for a patient/caregiver and view scheduled messages via the OWA UI.
- **Depends on**: OpenMRS platform 2.2.0; uses `uiframework`, `reporting`, `webservicesRest`, `metadatasharing`,
  `metadatadeploy`, `event`, `calculation`, `serialization.xstream` modules (see `pom.xml` dependencyManagement).
  Built against the `referenceapplication` 2.4 distro BOM.
- **Part of distribution**: Connect for Life, built on OpenMRS — SMS/IVR/WhatsApp-based patient engagement for
  HIV/TB/vaccination programs. Repo: `github.com/johnsonandjohnson/openmrs-module-messages`.

## Stack

- Backend: Java 8, Maven, OpenMRS module framework (produces a `.omod` file), multi-module (`api`, `omod`, `owa`)
- Frontend: Open Web App (OWA) under `owa/` — React + Redux (sagas, reducers, selectors), TypeScript config present,
  Ant Design (`antd`), webpack build, Jest + Enzyme for tests
- Build tool: Maven for `api`/`omod`; `owa` build is wired into the Maven reactor via a Maven plugin that shells out
  to npm/webpack (see `owa/pom.xml`, `owa/webpack.config.js`)

## Build & run

```bash
# Standard build (includes static analysis + OWA npm/webpack build)
mvn clean install

# Skip tests
mvn clean install -DskipTests

# Skip the OWA/frontend build during iteration (faster; UI must have been built once already,
# and the built zip is not deleted on `clean` when this profile is used)
mvn clean install -P no-npm

# Skip static analysis (checkstyle/PMD/findbugs) during iteration
mvn clean install -P dev

# Combine both for fastest inner-loop builds
mvn clean install -P dev,no-npm

# Deploy directly into a running local OpenMRS install (updates web resources without reinstalling)
mvn package -P deploy-web -D deploy.path="../../openmrs-1.8.x/webapp/src/main/webapp"

# Code coverage (JaCoCo reports at api/target/site/jacoco/index.html and omod/target/site/jacoco/index.html)
mvn clean install -P code-coverage
```

- OpenMRS SDK setup (only needed once per machine):
  `mvn org.openmrs.maven.plugins:openmrs-sdk-maven-plugin:setup-sdk`
- Java version required: **1.8** (`javaCompilerSource`/`javaCompilerTarget` = 1.8; CI uses Temurin JDK 8)
- Alternative deploy: copy `omod/target/*.omod` into `~/.OpenMRS/modules` and restart Tomcat/OpenMRS. If uploads
  via the admin UI are disabled, this is the only path.

## Tests

```bash
mvn test
```
- CI (`.github/workflows/maven.yml`) runs `mvn -B package --file pom.xml` on JDK 8 for pushes/PRs to `main`.
- The full `mvn package`/`install` build runs checkstyle, PMD, and findbugs by default — use the `dev` profile to
  skip them while iterating, but run a full build before pushing since CI does not skip them.
- OWA has its own Jest/Enzyme test suite (`owa/karma.conf.js`, Jest config in `owa/package.json`) that runs as part
  of the npm build step unless `-P no-npm` is used.

## Project structure

```
api/        - Java service layer: domain/model classes, DAOs, services (MessagingService, TemplateService,
              MessagesSchedulerService, ITR* services, etc.), scheduler/execution logic, event handling,
              validators, DTOs and mappers — see api/src/main/java/org/openmrs/module/messages/{domain,api}/
omod/       - Module activator, controllers, fragments (patientdashboard, patientheader), legacy JSP/webapp
              resources (styles/scripts) — see omod/src/main/webapp/fragments/
owa/        - Open Web App frontend: app/js/{components,reducers,sagas,selectors,config,shared}, built via
              webpack and packaged into the module
```

## Known gotchas

- The OWA/npm build is the slow part of `mvn install` — during active backend iteration use `-P no-npm` (requires
  the UI to have been built at least once already).
- `-P dev` disables checkstyle/PMD/findbugs; don't rely on it as a substitute for a real check before pushing —
  CI runs the full `mvn package` without that profile.
- Commit messages/PRs in this repo are prefixed with a JIRA ticket number (e.g. `3324: Small refactor...`) —
  follow that convention when writing commit messages.
- `messages.defaultUserTimezone` and patient-timezone handling for scheduled execution start times has been a
  recurring source of bugs (see recent commits around ticket 3310) — be careful with timezone conversions in
  `ScheduledExecutionContext`/execution code.

## OpenMRS domain notes

- Core domain model (in `api/.../api/model`): `Message`, `Template`, `NotificationTemplate`, `TemplateField`,
  `TemplateFieldValue`/`TemplateFieldDefaultValue`, `PatientTemplate`, `ScheduledService`/`ScheduledServiceGroup`,
  `ScheduledExecutionContext`, `Actor`/`ActorType`, `ActorResponse`/`ActorResponseType`, `DeliveryAttempt`,
  `PersonStatus`, `AdherenceFeedback`, `CountryProperty`, `GraphConfig`.
- Key services: `MessagingService`, `MessagesSchedulerService`, `MessagesExecutionService`,
  `MessagesDeliveryService`, `TemplateService`/`TemplateFieldService`, `PatientTemplateService`, `ActorService`,
  `HealthTipService`, `NotificationTemplateService`, ITR (interactive-text-response?) integration services
  (`ITRService`, `ITRConverterService`, `ITRScriptUtilsService`, `ITRMessageSenderService`).
- Relevant docs: OpenMRS Wiki (wiki.openmrs.org), OpenMRS SDK docs.

## Conventions

- Static analysis config lives at repo root: `checkstyle.xml`, `pmd.xml`, `findbugs-include.xml` — all enforced
  during a normal `mvn package`/`install`.
- Commit/PR titles start with the JIRA ticket number, e.g. `3324: Small refactor of method for getting patient
  template (#18)`.
- Issues/tickets are tracked in JIRA (ticket numbers referenced in commits), not GitHub Issues.

## When debugging

- Always state which module (`api`, `omod`, or `owa`) and which command produced the error.
- Prefer exploring the actual source over guessing from a stack trace alone.
- If you discover a new gotcha, add it to the "Known gotchas" section above.
