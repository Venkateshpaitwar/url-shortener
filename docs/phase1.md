## Known Issue: JVM timezone alias breaks Postgres connection

**Symptom:** Build failed with a misleading error:
Unable to determine Dialect without JDBC metadata

The real cause was buried one level deeper:
FATAL: invalid value for parameter "TimeZone": "Asia/Calcutta"

**Root cause:** The Surefire test JVM reported `user.timezone=Asia/Calcutta`,
a legacy Java timezone alias. PostgreSQL's JDBC driver passes this along at
connection time, and Postgres doesn't recognize the old alias — only
`Asia/Kolkata`. Since the connection never succeeded, Hibernate couldn't
read JDBC metadata, and defaulted to a confusing "can't determine dialect"
error instead of surfacing the real timezone rejection.

**Debugging process:**
1. Verified Postgres itself: `SHOW timezone;` → `Etc/UTC` (fine)
2. Verified Windows OS timezone → `India Standard Time` (fine)
3. Checked env vars (`TZ`, `JAVA_TOOL_OPTIONS`, `MAVEN_OPTS`) → clean
4. Found the actual culprit: the test JVM's `user.timezone` system property
5. Fixed via Maven Surefire config

**Fix:** Pin the test JVM's timezone explicitly in `pom.xml`:
```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <configuration>
    <argLine>-Duser.timezone=Asia/Kolkata</argLine>
  </configuration>
</plugin>
```

**Lesson:** Java's timezone ID list still contains deprecated aliases
(e.g. `Asia/Calcutta` → `Asia/Kolkata`) that some JVMs/locales resolve to
by default. When a downstream service rejects a value your app didn't
explicitly set, check where that value is actually coming from — the
JVM's default state, not just your own config files.