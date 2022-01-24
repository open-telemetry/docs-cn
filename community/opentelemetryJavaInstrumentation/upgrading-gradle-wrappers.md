# 升级gradle 包装器

设置`GRADLE_VERSION`为gradle的版本.

设置`GRADLE_VERSION_CHECKSUM` 为 "Binary-only (-bin) ZIP Checksum" ，"Binary-only (-bin) ZIP Checksum"可以从 https://gradle.org/release-checksums/ 中获取。

然后运行:

```
for dir in . \
           benchmark-overhead \
           examples/distro \
           examples/extension \
           smoke-tests/images/fake-backend \
           smoke-tests/images/grpc \
           smoke-tests/images/quarkus \
           smoke-tests/images/servlet \
           smoke-tests/images/play \
           smoke-tests/images/spring-boot
do
  (cd $dir && ./gradlew wrapper --gradle-version $GRADLE_VERSION \
                  --gradle-distribution-sha256-sum=$GRADLE_VERSION_CHECKSUM)
done
```
