---
layout: post
title:  "33. Vấn đề format code trong team."
date:   2023-03-05 00:01:00 +0000
category: technical
---
## 1. Vấn đề 
Space indentation giữa các IDE hay code style là khác nhau. Sự khác biệt về indentation gây ra khó khăn khi review Pull Request hay sự thay đổi các file không cần thiết khi commit code. Thêm vào đó, mỗi team, mỗi projet có quy định về code style riêng. Một người làm việc với nhiều project, sẽ rất có khó trong việc tuân thủ tất cả chuẩn về coding style. Việc thay đổi code style theo project một cách thủ là không hiệu quả. Hay nói tóm lại là tôi lười khi cứ phải thay đổi liên tục. 

##  2. Giải pháp 
Thay vì phải config coding style trên IDE, ta sẽ dùng các plugin của các công cụ build project như Maven, để nó tự động format code theo 1 dạng quy định sẵn khi commit code. 

Ta muốn quá trình format này được diễn ra một cách tự động khi commit code, không cần phải chạy lệnh thủ công. Để làm được điều này, ta có thể dùng git hooks. 

Lưu ý rằng, sau khi thay đổi format của file, ta cần phải add lại những file này vào git staging bằng lệnh git add. Để cho dễ dàng, ta sẽ viết một đoạn script, liệt kê những file đã được add vào git staging, format chúng và rồi add lại vào git staging.

## 3. Chi tiết thực hiện 
### 3.1 Tạo git-hooks folder trong project 
Trong folder này, tạo file pre-commit. (Nhớ cấp quyền execution cho file này). Khi chạy lệnh `mvn clean install`, file pre-commit sẽ được copy vào folder `.git/hooks` của project. Lúc này, pre-commit mới được thực thi khi ta gõ lệnh `git commit`. 

Nội dụng file pre-commit 

```bash
#!/bin/sh
FILES=$(git diff --cached --name-only --diff-filter=ACMR | sed 's| |\\ |g')
[ -z "$FILES" ] && exit 0

# Prettify all selected files
for FILE in $FILES
do
  mvn prettier:write -Dprettier.inputGlobs=$FILE
done
# Add back the modified/prettified files to staging
echo "$FILES" | xargs git add

exit 0

```

### 3.2 Thêm plugins vào pom.xml
Plugin để format java code. Nếu muốn gọi plugin này bằng cmd, gõ `mvn prettier:write`. Lệnh này sẽ format toàn bộ file trong project. Xem thêm tại: <https://github.com/HubSpot/prettier-maven-plugin> 

```xml
<plugin>
	<groupId>com.hubspot.maven.plugins</groupId>
	<artifactId>prettier-maven-plugin</artifactId>
	<version>0.16</version>
	<configuration>
		<prettierJavaVersion>1.5.0</prettierJavaVersion>
		<printWidth>90</printWidth>
		<tabWidth>2</tabWidth>
		<useTabs>false</useTabs>
		<ignoreConfigFile>true</ignoreConfigFile>
		<ignoreEditorConfig>true</ignoreEditorConfig>
	</configuration>
</plugin>
```

Để copy file pre-commit vào .git/hooks, dùng maven-resources-plugin

```xml  
<plugin>
	<artifactId>maven-resources-plugin</artifactId>
	<version>3.2.0</version>
	<executions>
		<execution>
			<id>copy-hooks</id>
			<phase>validate</phase>
			<goals>
				<goal>copy-resources</goal>
			</goals>
			<configuration>
				<outputDirectory>${basedir}/.git/hooks</outputDirectory>
				<resources>
					<resource>
						<directory>git-hooks</directory>
						<includes>
							<include>pre-commit</include>
						</includes>
					</resource>
				</resources>
			</configuration>
		</execution>
	</executions>
</plugin>
```

Về quy tắc gọi plugins, tham khảo thêm tại: <https://maven.apache.org/guides/introduction/introduction-to-plugin-prefix-mapping.html>

### 3.3 Lưu ý 
Thực thi lệnh `mvn clean install` trước để copy file pre-commit vào `.git/hooks`. Sau đó, mỗi lần commit, plugins prettier sẽ tự động format code những file đã thay đổi. Điều này giúp thành viên trong team tự do trong việc chọn coding style, giảm bớt 1 quy định trong team. Với tôi, càng ít quy định, làm việc càng khỏe. 

Để tiện, tôi chuẩn bị demo tại link: <https://github.com/NhatVu/spring-boot-config> 
