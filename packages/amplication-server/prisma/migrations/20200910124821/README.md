# Migration `20200910124821`

This migration has been generated by Yuval Hazaz at 9/10/2020, 3:48:21 PM.
You can check out the [state of the schema](./schema.prisma) after the migration.

## Database Steps

```sql
DROP INDEX "public"."EntityPermissionField.entityPermissionId_fieldPermanentId_uniqu"

ALTER TABLE "public"."EntityPermissionField" DROP CONSTRAINT "EntityPermissionField_entityPermissionId_fkey"

ALTER TABLE "public"."Build" ADD COLUMN "version" text   NOT NULL ,
ADD COLUMN "message" text   

ALTER TABLE "public"."EntityField" ADD COLUMN "position" integer   

ALTER TABLE "public"."EntityPermissionField" DROP COLUMN "entityPermissionId",
ADD COLUMN "permissionId" text   NOT NULL 

CREATE UNIQUE INDEX "Build.appId_version_unique" ON "public"."Build"("appId", "version")

CREATE UNIQUE INDEX "EntityPermissionField.permissionId_fieldPermanentId_unique" ON "public"."EntityPermissionField"("permissionId", "fieldPermanentId")

ALTER TABLE "public"."EntityPermissionField" ADD FOREIGN KEY ("permissionId")REFERENCES "public"."EntityPermission"("id") ON DELETE CASCADE ON UPDATE CASCADE
```

## Changes

```diff
diff --git schema.prisma schema.prisma
migration ..20200910124821
--- datamodel.dml
+++ datamodel.dml
@@ -1,0 +1,328 @@
+datasource db {
+  provider = "postgresql"
+  url = "***"
+}
+
+generator client {
+  provider        = "prisma-client-js"
+  previewFeatures = ["insensitiveFilters"]
+
+}
+
+// generator typegraphql {
+//   provider = "node ./node_modules/typegraphql-prisma/generator.js"
+//   output   = "./dal"
+// }
+
+model Account {
+  id            String   @default(cuid()) @id
+  createdAt     DateTime @default(now())
+  updatedAt     DateTime @updatedAt
+  email         String   @unique
+  firstName     String
+  lastName      String
+  password      String
+  users         User[]   @relation("AccountOnUser")
+  currentUser   User?    @relation(fields: [currentUserId], references: [id])
+  currentUserId String?
+  githubId      String?
+}
+
+model Organization {
+  id              String   @default(cuid()) @id
+  createdAt       DateTime @default(now())
+  updatedAt       DateTime @updatedAt
+  name            String   @unique
+  defaultTimeZone String
+  address         String
+  apps            App[]
+  users           User[]
+}
+
+model User {
+  id             String       @default(cuid()) @id
+  createdAt      DateTime     @default(now())
+  updatedAt      DateTime     @updatedAt
+  account        Account      @relation("AccountOnUser", fields: [accountId], references: [id])
+  accountId      String
+  organization   Organization @relation(fields: [organizationId], references: [id])
+  organizationId String
+  userRoles      UserRole[]
+  Account        Account[]
+
+  @@unique([accountId, organizationId])
+  commits       Commit[]
+  lockedEntitis Entity[]
+  builds        Build[]
+}
+
+// enum Role {
+// ADMIN
+// USER
+// ORGANIZATION_ADMIN
+// PROJECT_ADMIN
+// }
+model UserRole {
+  id        String   @default(cuid()) @id
+  createdAt DateTime @default(now())
+  updatedAt DateTime @updatedAt
+  user      User     @relation(fields: [userId], references: [id])
+  userId    String
+  role      String
+
+  @@unique([userId, role])
+}
+
+model App {
+  id             String       @default(cuid()) @id
+  createdAt      DateTime     @default(now())
+  updatedAt      DateTime     @updatedAt
+  organization   Organization @relation(fields: [organizationId], references: [id])
+  organizationId String
+  name           String
+  description    String
+  entities       Entity[]
+  blocks         Block[]
+  appRoles       AppRole[]
+  commits        Commit[]
+  builds         Build[]
+
+  @@unique([organizationId, name])
+}
+
+model AppRole {
+  id          String   @default(cuid()) @id
+  createdAt   DateTime @default(now())
+  updatedAt   DateTime @updatedAt
+  app         App      @relation(fields: [appId], references: [id])
+  appId       String
+  name        String
+  displayName String
+  description String?
+
+  @@unique([appId, name])
+  @@unique([appId, displayName])
+  entityPermissionRoles EntityPermissionRole[]
+}
+
+model Commit {
+  id        String   @default(cuid()) @id
+  createdAt DateTime @default(now())
+  app       App      @relation(fields: [appId], references: [id])
+  appId     String
+  user      User     @relation(fields: [userId], references: [id])
+  userId    String
+  message   String
+
+  entityVersions EntityVersion[]
+}
+
+model Entity {
+  id                String          @default(cuid()) @id
+  createdAt         DateTime        @default(now())
+  updatedAt         DateTime        @updatedAt
+  app               App             @relation(fields: [appId], references: [id])
+  appId             String
+  name              String
+  displayName       String
+  pluralDisplayName String
+  description       String?
+  entityVersions    EntityVersion[]
+  lockedByUser      User?           @relation(fields: [lockedByUserId], references: [id])
+  lockedByUserId    String?
+  lockedAt          DateTime?
+  deletedAt         DateTime?
+
+  @@unique([appId, name])
+  @@unique([appId, pluralDisplayName])
+  @@unique([appId, displayName])
+}
+
+model EntityVersion {
+  id                String             @default(cuid()) @id
+  createdAt         DateTime           @default(now())
+  updatedAt         DateTime           @updatedAt
+  entity            Entity             @relation(fields: [entityId], references: [id])
+  entityId          String
+  versionNumber     Int
+  name              String
+  displayName       String
+  pluralDisplayName String
+  description       String?
+  entityFields      EntityField[]
+  commit            Commit?            @relation(fields: [commitId], references: [id])
+  commitId          String?
+  entityPermissions EntityPermission[]
+  builds            Build[]            @relation(references: [id])
+  deleted           Boolean?
+
+  @@unique([entityId, versionNumber])
+
+}
+
+enum EnumEntityAction {
+  View
+  Create
+  Update
+  Delete
+  Search
+}
+
+enum EnumEntityPermissionType {
+  AllRoles
+  Granular
+  Disabled
+}
+
+model EntityPermission {
+  id               String                   @default(cuid()) @id
+  entityVersion    EntityVersion            @relation(fields: [entityVersionId], references: [id])
+  entityVersionId  String
+  action           EnumEntityAction
+  type             EnumEntityPermissionType
+  permissionRoles  EntityPermissionRole[]
+  permissionFields EntityPermissionField[]
+  @@unique([entityVersionId, action])
+}
+
+model EntityPermissionRole {
+  id                   String                  @default(cuid()) @id
+  permission           EntityPermission        @relation(fields: [entityVersionId, action], references: [entityVersionId, action])
+  //entityPermissionId   String
+  entityVersionId      String
+  action               EnumEntityAction
+  appRole              AppRole                 @relation(fields: [appRoleId], references: [id])
+  appRoleId            String
+  permissionFieldRoles EntityPermissionField[]
+  @@unique([entityVersionId, action, appRoleId])
+
+}
+
+model EntityPermissionField {
+  id                   String                 @default(cuid()) @id
+  permission           EntityPermission       @relation(fields: [permissionId], references: [id])
+  permissionId         String
+  field                EntityField            @relation(fields: [fieldPermanentId, entityVersionId], references: [fieldPermanentId, entityVersionId])
+  fieldPermanentId     String
+  entityVersionId      String
+  permissionFieldRoles EntityPermissionRole[]
+
+  @@unique([permissionId, fieldPermanentId])
+}
+
+model EntityField {
+  id               String        @default(cuid()) @id
+  createdAt        DateTime      @default(now())
+  updatedAt        DateTime      @updatedAt
+  entityVersion    EntityVersion @relation(fields: [entityVersionId], references: [id])
+  entityVersionId  String
+  fieldPermanentId String        @default(cuid())
+  name             String
+  displayName      String
+  dataType         EnumDataType
+  properties       Json
+  required         Boolean
+  // TBD
+  searchable       Boolean
+  description      String
+  position         Int?
+
+  @@unique([entityVersionId, fieldPermanentId])
+  @@unique([entityVersionId, name])
+  @@unique([entityVersionId, displayName])
+  EntityPermissionField EntityPermissionField[]
+}
+
+enum EnumDataType {
+  SingleLineText
+  MultiLineText
+  Email
+  AutoNumber
+  WholeNumber
+  DateTime
+  DecimalNumber
+  Lookup
+  MultiSelectOptionSet
+  OptionSet
+  Boolean
+  GeographicAddress
+  Id
+  CreatedAt
+  UpdatedAt
+}
+
+model Block {
+  id            String         @default(cuid()) @id
+  createdAt     DateTime       @default(now())
+  updatedAt     DateTime       @updatedAt
+  app           App            @relation(fields: [appId], references: [id])
+  appId         String
+  parentBlock   Block?         @relation(fields: [parentBlockId], references: [id])
+  parentBlockId String?
+  blockType     EnumBlockType
+  displayName   String
+  description   String?
+  blockVersions BlockVersion[]
+
+  @@unique([appId, displayName])
+  blocks Block[] @relation("BlockToBlock")
+}
+
+enum EnumBlockType {
+  AppSettings
+  Flow
+  ConnectorRestApi
+  ConnectorRestApiCall
+  ConnectorSoapApi
+  ConnectorFile
+  EntityApi
+  EntityApiEndpoint
+  FlowApi
+  Layout
+  CanvasPage
+  EntityPage
+  Document
+}
+
+model BlockVersion {
+  id               String   @default(cuid()) @id
+  createdAt        DateTime @default(now())
+  updatedAt        DateTime @updatedAt
+  block            Block    @relation(fields: [blockId], references: [id])
+  blockId          String
+  versionNumber    Int
+  label            String
+  inputParameters  Json? //JSON
+  outputParameters Json? //JSON
+  settings         Json //JSON
+  builds           Build[]  @relation(references: [id])
+
+  @@unique([blockId, versionNumber])
+}
+
+enum EnumBuildStatus {
+  Completed
+  Waiting
+  Active
+  Delayed
+  Failed
+  Paused
+}
+
+model Build {
+  id             String          @default(cuid()) @id
+  createdAt      DateTime        @default(now())
+  createdBy      User            @relation(fields: [userId], references: [id])
+  app            App             @relation(fields: [appId], references: [id])
+  appId          String
+  userId         String
+  status         EnumBuildStatus
+  blockVersions  BlockVersion[]  @relation(references: [id])
+  entityVersions EntityVersion[] @relation(references: [id])
+  version        String
+  message        String?
+
+
+  @@unique([appId, version])
+
+}
```

