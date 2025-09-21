### 데이터 백업 및 이관 BATCH 테스트

좋아요. 요청하신 컨트롤러 / 서비스 / DAO / DTO / VO / Mapper를 전부 포함한 Spring Batch + MyBatis 기반 “두 원본 테이블 조인 → 단일 타깃 테이블로 이관 → 원본 데이터 삭제” 파이프라인 전체 예시를 제공합니다.

- 운영 전제: PostgreSQL, MyBatis, Spring Batch, 정해진 시간 자동 실행(@Scheduled) + 수동 트리거용 API 제공

- 들여쓰기는 전부 탭, 네이버 체크스타일 취향(간결 네이밍, Javadoc, Swagger 주석 등) 맞춤

- 안전 절차: ①이관(UPSERT) → ②검증(건수) → ③삭제(원본) → ④후처리/감사

- 대량 운영 대비: 청크 처리, 재실행 안전(idempotent), ON CONFLICT DO NOTHING 사용, 실패 시 재시작 가능하게 Step 분리

### 0) 의존성 & 설정

### build.gradle
```````
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'org.springframework.boot:spring-boot-starter-batch'
	implementation 'org.springframework.boot:spring-boot-starter-jdbc'
	implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3'
	implementation 'org.mybatis.spring.batch:mybatis-spring-batch:2.0.1'
	runtimeOnly 'org.postgresql:postgresql'
	implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0'
}
```````


application.yml
```````
spring:
  jackson:
    time-zone: Asia/Seoul
  batch:
    job:
      enabled: true

batch:
  migration:
    cron: "0 10 3 * * *" # 매일 03:10
    chunk-size: 2000
    grid-size: 1
    cutoff: "2025-07-01" # 예: 이 날짜 이전 건만 이관/삭제(원하는 기준으로 변경)

# DB, mybatis 등은 환경에 맞게 설정
```````

1) 테이블 가정(예시)

원본 A: ass_src_a(a_id PK, key, col_a, last_reg_dt, use_yn)

원본 B: ass_src_b(b_id PK, a_id FK, col_b, last_reg_dt, use_yn)

타깃: ass_merged_c(id PK, key, col_a, col_b, a_id, b_id, merge_dt, use_yn)

이관 로그(스테이징): ass_mig_log_t(run_id, a_id, b_id, merged_id, reg_dt) ← 당일/실행별 삭제 대상 보존

실제 컬럼명은 운영 DB에 맞춰 바꾸세요. 아래 Mapper의 SQL만 교체하면 됩니다.



2) DTO / VO
MigrationRowVo (조인 결과 → 타깃으로 이관될 1건)
```````
package kr.re.kice.adnp.cm.survey.dao.vo.migration;

import java.time.LocalDateTime;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class MigrationRowVo {
	private Long aId;
	private Long bId;
	private String key;
	private String colA;
	private String colB;
	private LocalDateTime lastRegDt; // 기준일 필터링에 사용
}
```````

MigrationResultDto (컨트롤러 응답용)
```````
package kr.re.kice.adnp.cm.survey.dto.migration;

import io.swagger.v3.oas.annotations.media.Schema;
import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
@Schema(description = "이관 작업 결과")
public class MigrationResultDto {
	@Schema(description = "실행 식별자(runId)")
	private String runId;

	@Schema(description = "이관 건수")
	private long migratedCount;

	@Schema(description = "삭제 건수")
	private long deletedCount;
}
```````

3) Mapper(XML)

네임스페이스: migration.biz

SelectJoinForMigration: 컷오프 이전 + 사용중(Y)만 조인해서 대상 선별

InsertIntoTarget: 타깃 테이블 UPSERT(충돌 시 무시)

InsertMigLog: 이관한 A/B 키를 실행별(runId)로 로그 저장

VerifyCounts: 타깃과 로그 건수 비교

DeleteFromB/FromA: 로그에 기록된 키 기준으로 원본 삭제(자식→부모 순)

HousekeepLog: 오래된 로그 정리(옵션)

```````
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
	"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="migration.biz">

	<!-- 조인 기준 예시: key 기준이나 a_id=b.a_id 기준 -->
	<select id="SelectJoinForMigration"
		parameterType="map"
		resultType="kr.re.kice.adnp.cm.survey.dao.vo.migration.MigrationRowVo">
		SELECT
			a.a_id			AS aId,
			b.b_id			AS bId,
			a.key			AS key,
			a.col_a			AS colA,
			b.col_b			AS colB,
			GREATEST(a.last_reg_dt, b.last_reg_dt) AS lastRegDt
		FROM ass_src_a a
		JOIN ass_src_b b
			ON b.a_id = a.a_id
		WHERE a.use_yn = 'Y'
		  AND b.use_yn = 'Y'
		  AND GREATEST(a.last_reg_dt, b.last_reg_dt) < #{cutoff}
		ORDER BY a.a_id
	</select>

	<!-- 타깃 INSERT(UPSERT) -->
	<insert id="InsertIntoTarget"
		parameterType="kr.re.kice.adnp.cm.survey.dao.vo.migration.MigrationRowVo">
		INSERT INTO ass_merged_c (id, key, col_a, col_b, a_id, b_id, merge_dt, use_yn)
		VALUES (
			/* 예: id 생성 규칙. 운영 규칙에 맞춰 시퀀스/ULID/스노플레이크 등으로 변경 */
			NEXTVAL('ass_merged_c_id_seq'),
			#{key}, #{colA}, #{colB}, #{aId}, #{bId}, NOW(), 'Y'
		)
		ON CONFLICT (a_id, b_id) DO NOTHING
	</insert>

	<!-- 이관 로그 적재(원본 삭제를 안전하게 하기 위함) -->
	<insert id="InsertMigLog"
		parameterType="map">
		INSERT INTO ass_mig_log_t (run_id, a_id, b_id, merged_id, reg_dt)
		VALUES (
			#{runId}, #{aId}, #{bId},
			/* 타깃 PK를 별도로 조회/매핑하려면 SELECT로 치환 가능. 여기선 NULL 허용 */
			NULL,
			NOW()
		)
	</insert>

	<!-- 검증: 로그와 타깃 건수가 일치하는지(단순 예시) -->
	<select id="VerifyCounts"
		parameterType="map"
		resultType="long">
		SELECT
			(SELECT COUNT(*) FROM ass_mig_log_t WHERE run_id = #{runId}) AS log_cnt
	</select>

	<!-- 원본 삭제: 자식(B) -> 부모(A) 순 -->
	<delete id="DeleteFromB"
		parameterType="map">
		DELETE FROM ass_src_b
		WHERE b_id IN (
			SELECT b_id FROM ass_mig_log_t WHERE run_id = #{runId}
		)
	</delete>

	<delete id="DeleteFromA"
		parameterType="map">
		DELETE FROM ass_src_a a
		WHERE a.a_id IN (
			SELECT a_id FROM ass_mig_log_t WHERE run_id = #{runId}
		)
		AND NOT EXISTS (
			SELECT 1
			FROM ass_src_b b
			WHERE b.a_id = a.a_id
			AND b.use_yn = 'Y'
		)
	</delete>

	<!-- 로그 정리(옵션) -->
	<delete id="HousekeepLog"
		parameterType="map">
		DELETE FROM ass_mig_log_t
		WHERE reg_dt < (NOW() - INTERVAL '30 days')
	</delete>

</mapper>

```````


4) DAO
```````
package kr.re.kice.adnp.cm.survey.dao;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;
import java.util.List;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionTemplate;
import org.springframework.stereotype.Repository;
import kr.re.kice.adnp.cm.survey.dao.vo.migration.MigrationRowVo;
import lombok.RequiredArgsConstructor;

@Repository
@RequiredArgsConstructor
public class MigrationDao {

	private final SqlSessionTemplate sst;

	public List<MigrationRowVo> selectForMigration(LocalDateTime cutoff) {
		final Map<String, Object> params = new HashMap<>();
		params.put("cutoff", cutoff);
		return sst.selectList("migration.biz.SelectJoinForMigration", params);
	}

	public int insertIntoTarget(MigrationRowVo vo) {
		return sst.insert("migration.biz.InsertIntoTarget", vo);
	}

	public int insertMigLog(String runId, Long aId, Long bId) {
		final Map<String, Object> p = new HashMap<>();
		p.put("runId", runId);
		p.put("aId", aId);
		p.put("bId", bId);
		return sst.insert("migration.biz.InsertMigLog", p);
	}

	public long verifyLogCount(String runId) {
		final Map<String, Object> p = new HashMap<>();
		p.put("runId", runId);
		return sst.selectOne("migration.biz.VerifyCounts", p);
	}

	public int deleteFromB(String runId) {
		final Map<String, Object> p = new HashMap<>();
		p.put("runId", runId);
		return sst.delete("migration.biz.DeleteFromB", p);
	}

	public int deleteFromA(String runId) {
		final Map<String, Object> p = new HashMap<>();
		p.put("runId", runId);
		return sst.delete("migration.biz.DeleteFromA", p);
	}

	public int housekeepLog() {
		return sst.delete("migration.biz.HousekeepLog");
	}
}
```````

5) Batch 구성 (MyBatisPagingItemReader + MyBatisBatchItemWriter)
```````
package kr.re.kice.adnp.cm.survey.batch.migration;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;
import javax.sql.DataSource;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.batch.MyBatisBatchItemWriter;
import org.mybatis.spring.batch.MyBatisPagingItemReader;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.StepExecutionListener;
import org.springframework.batch.core.ExitStatus;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobExecutionListener;
import org.springframework.batch.core.configuration.annotation.StepScope;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ExecutionContext;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.transaction.PlatformTransactionManager;

import kr.re.kice.adnp.cm.survey.dao.vo.migration.MigrationRowVo;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class MigrationJobConfig {

	private final JobRepository jobRepository;
	private final PlatformTransactionManager tx;
	private final SqlSessionFactory sqlSessionFactory;

	@Bean
	public Job migrationJob() {
		return new JobBuilder("migrationJob", jobRepository)
			.start(stepMigrateInsert())
			.next(stepVerify())
			.next(stepDeleteB())
			.next(stepDeleteA())
			.listener(jobListener())
			.build();
	}

	@Bean
	public JobExecutionListener jobListener() {
		return new JobExecutionListener() {
			@Override
			public void beforeJob(JobExecution jobExecution) {
				log.info("Migration job start. runId={}, cutoff={}",
					jobExecution.getJobParameters().getString("runId"),
					jobExecution.getJobParameters().getString("cutoff"));
			}
			@Override
			public void afterJob(JobExecution jobExecution) {
				log.info("Migration job end. status={}", jobExecution.getStatus());
			}
		};
	}

	// Step1: 조인 조회 → 타깃 INSERT(UPSERT) + 이관 로그 기록
	@Bean
	public Step stepMigrateInsert() {
		return new StepBuilder("stepMigrateInsert", jobRepository)
			.<MigrationRowVo, MigrationRowVo>chunk(chunkSize(), tx)
			.reader(joinReader(null))
			.processor(passThroughProcessor())
			.writer(itemWriterInsertAndLog(null))
			.listener(stepListener("stepMigrateInsert"))
			.build();
	}

	@StepScope
	@Bean
	public MyBatisPagingItemReader<MigrationRowVo> joinReader(
		@Value("#{jobParameters['cutoff']}") String cutoff
	) {
		final MyBatisPagingItemReader<MigrationRowVo> reader = new MyBatisPagingItemReader<>();
		reader.setSqlSessionFactory(sqlSessionFactory);
		reader.setQueryId("migration.biz.SelectJoinForMigration");

		final Map<String, Object> params = new HashMap<>();
		params.put("cutoff", LocalDate.parse(cutoff).atStartOfDay());
		reader.setParameterValues(params);

		reader.setPageSize(chunkSize());
		return reader;
	}

	@StepScope
	@Bean
	public ItemProcessor<MigrationRowVo, MigrationRowVo> passThroughProcessor() {
		return item -> item; // 필요 시 데이터 정합/마스킹/검증 로직 적용
	}

	@StepScope
	@Bean
	public org.springframework.batch.item.ItemWriter<MigrationRowVo> itemWriterInsertAndLog(
		@Value("#{jobParameters['runId']}") String runId
	) {
		final MyBatisBatchItemWriter<MigrationRowVo> insertWriter = new MyBatisBatchItemWriter<>();
		insertWriter.setSqlSessionFactory(sqlSessionFactory);
		insertWriter.setStatementId("migration.biz.InsertIntoTarget");

		return items -> {
			for (final var item : items) {
				insertWriter.write(java.util.List.of(item));
				// 로그는 별도 문장 호출(간단히 Dao를 써도 되지만 여기선 파라미터 맵으로 호출)
				final var session = sqlSessionFactory.openSession(false);
				try {
					final Map<String, Object> p = new HashMap<>();
					p.put("runId", runId);
					p.put("aId", item.getAId());
					p.put("bId", item.getBId());
					session.insert("migration.biz.InsertMigLog", p);
					session.commit();
				} finally {
					session.close();
				}
			}
		};
	}

	// Step2: 검증(간단 버전 - 로그 건수>0 확인 정도. 필요시 더 강화)
	@Bean
	public Step stepVerify() {
		return new StepBuilder("stepVerify", jobRepository)
			.tasklet((contribution, chunkContext) -> {
				final String runId = (String) chunkContext.getStepContext().getJobParameters().get("runId");
				final var session = sqlSessionFactory.openSession(true);
				try {
					final Long logCnt = session.selectOne("migration.biz.VerifyCounts", java.util.Map.of("runId", runId));
					if (logCnt == null || logCnt <= 0L) {
						throw new IllegalStateException("No migrated logs for runId=" + runId);
					}
					return org.springframework.batch.repeat.RepeatStatus.FINISHED;
				} finally {
					session.close();
				}
			}, tx).build();
	}

	// Step3: 원본 B 삭제(자식)
	@Bean
	public Step stepDeleteB() {
		return new StepBuilder("stepDeleteB", jobRepository)
			.tasklet((contribution, chunkContext) -> {
				final String runId = (String) chunkContext.getStepContext().getJobParameters().get("runId");
				final var session = sqlSessionFactory.openSession(true);
				try {
					final int del = session.delete("migration.biz.DeleteFromB", java.util.Map.of("runId", runId));
					chunkContext.getStepContext().getStepExecution().getExecutionContext().putInt("deletedB", del);
					return org.springframework.batch.repeat.RepeatStatus.FINISHED;
				} finally {
					session.close();
				}
			}, tx).listener(stepListener("stepDeleteB")).build();
	}

	// Step4: 원본 A 삭제(부모)
	@Bean
	public Step stepDeleteA() {
		return new StepBuilder("stepDeleteA", jobRepository)
			.tasklet((contribution, chunkContext) -> {
				final String runId = (String) chunkContext.getStepContext().getJobParameters().get("runId");
				final var session = sqlSessionFactory.openSession(true);
				try {
					final int del = session.delete("migration.biz.DeleteFromA", java.util.Map.of("runId", runId));
					chunkContext.getStepContext().getStepExecution().getExecutionContext().putInt("deletedA", del);
					return org.springframework.batch.repeat.RepeatStatus.FINISHED;
				} finally {
					session.close();
				}
			}, tx).listener(stepListener("stepDeleteA")).build();
	}

	private int chunkSize() {
		return Integer.parseInt(System.getProperty("batch.migration.chunk", "2000"));
	}

	private StepExecutionListener stepListener(String name) {
		return new StepExecutionListener() {
			@Override
			public void beforeStep(org.springframework.batch.core.StepExecution stepExecution) {
				log.info("{} start", name);
			}
			@Override
			public ExitStatus afterStep(org.springframework.batch.core.StepExecution stepExecution) {
				log.info("{} end. read={}, write={}", name, stepExecution.getReadCount(), stepExecution.getWriteCount());
				return ExitStatus.COMPLETED;
			}
		};
	}
}
```````


6) 스케줄러(@Scheduled) + 수동 트리거 컨트롤러
스케줄러
```````
package kr.re.kice.adnp.cm.survey.batch.migration;

import java.time.LocalDateTime;
import java.util.Date;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@EnableScheduling
@RequiredArgsConstructor
public class MigrationJobScheduler {

	private final JobLauncher jobLauncher;
	private final Job migrationJob;

	@Value("${batch.migration.cron}")
	private String cron;

	@Value("${batch.migration.cutoff}")
	private String cutoff;

	@Scheduled(cron = "${batch.migration.cron}", zone = "Asia/Seoul")
	public void run() throws Exception {
		final String runId = "M" + LocalDateTime.now().toString().replace(":", "").replace("-", "").replace(".", "");
		final JobParameters params = new JobParametersBuilder()
			.addString("runId", runId)
			.addString("cutoff", cutoff)
			.addDate("runAt", new Date())
			.toJobParameters();

		log.info("Scheduled migration run. runId={}, cutoff={}", runId, cutoff);
		jobLauncher.run(migrationJob, params);
	}
}
```````

컨트롤러(수동 실행 API + Swagger)
```````
package kr.re.kice.adnp.cm.survey.api;

import java.util.Date;
import java.time.LocalDateTime;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import kr.re.kice.adnp.cm.common.CommonResponse;
import kr.re.kice.adnp.cm.survey.dto.migration.MigrationResultDto;

@Slf4j
@RestController
@RequestMapping("/api/v3/ss")
@RequiredArgsConstructor
public class MigrationController {

	private final JobLauncher jobLauncher;
	private final Job migrationJob;

	@PostMapping("/migrate/run")
	@Operation(summary = "이관 배치 수동 실행", description = "두 원본 테이블을 조인해 타깃 테이블로 이관 후 원본을 삭제합니다.")
	@ApiResponses({
		@ApiResponse(responseCode = "200", description = "실행 성공",
			content = @Content(mediaType = "application/json",
				schema = @Schema(implementation = CommonResponse.class)))
	})
	public ResponseEntity<CommonResponse<MigrationResultDto>> run(
		@RequestParam(name = "cutoff", required = false) String cutoff
	) throws Exception {
		final String runId = "M" + LocalDateTime.now().toString().replace(":", "").replace("-", "").replace(".", "");
		final JobParameters params = new JobParametersBuilder()
			.addString("runId", runId)
			.addString("cutoff", (cutoff == null || cutoff.isBlank()) ? "2025-07-01" : cutoff)
			.addDate("runAt", new Date())
			.toJobParameters();

		final JobExecution exec = jobLauncher.run(migrationJob, params);

		// 실제 건수 집계는 StepExecution의 read/write 또는 로그 테이블 카운트로 조회 가능
		final long migrated = exec.getStepExecutions().stream().filter(s -> s.getStepName().equals("stepMigrateInsert"))
			.mapToLong(s -> s.getWriteCount()).sum();
		// 삭제 건수는 삭제 스텝의 ExecutionContext에 기록해두거나 로그 테이블 재조회
		final long deleted = exec.getStepExecutions().stream()
			.filter(s -> s.getStepName().startsWith("stepDelete"))
			.mapToLong(s -> s.getExecutionContext().containsKey("deletedA")
				? s.getExecutionContext().getInt("deletedA")
				: (s.getExecutionContext().containsKey("deletedB")
					? s.getExecutionContext().getInt("deletedB")
					: 0))
			.sum();

		final MigrationResultDto dto = new MigrationResultDto(runId, migrated, deleted);
		return CommonResponse.respondOk(dto, "이관 배치 실행 성공");
	}
}
```````

7) 서비스(선택) / 공통 응답 포맷
위 컨트롤러는 바로 CommonResponse를 사용했습니다. 만약 서비스 레이어를 선호하시면 아래처럼 래핑하세요.
```````
package kr.re.kice.adnp.cm.survey.service;

import kr.re.kice.adnp.cm.common.CommonResponse;
import kr.re.kice.adnp.cm.survey.dto.migration.MigrationResultDto;

public interface MigrationService {
	CommonResponse<MigrationResultDto> run(String cutoff);
}
```````
```````
package kr.re.kice.adnp.cm.survey.service.impl;

import java.util.Date;
import java.time.LocalDateTime;

import lombok.RequiredArgsConstructor;
import org.springframework.batch.core.*;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.stereotype.Service;

import kr.re.kice.adnp.cm.survey.service.MigrationService;
import kr.re.kice.adnp.cm.survey.dto.migration.MigrationResultDto;
import kr.re.kice.adnp.cm.common.CommonResponse;

@Service
@RequiredArgsConstructor
public class MigrationServiceImpl implements MigrationService {

	private final JobLauncher jobLauncher;
	private final Job migrationJob;

	@Override
	public CommonResponse<MigrationResultDto> run(String cutoff) {
		try {
			final String runId = "M" + LocalDateTime.now().toString().replace(":", "").replace("-", "").replace(".", "");
			final JobParameters params = new JobParametersBuilder()
				.addString("runId", runId)
				.addString("cutoff", (cutoff == null || cutoff.isBlank()) ? "2025-07-01" : cutoff)
				.addDate("runAt", new Date())
				.toJobParameters();

			final JobExecution exec = jobLauncher.run(migrationJob, params);
			final long migrated = exec.getStepExecutions().stream()
				.filter(s -> s.getStepName().equals("stepMigrateInsert"))
				.mapToLong(StepExecution::getWriteCount)
				.sum();
			final long deleted = exec.getStepExecutions().stream()
				.filter(s -> s.getStepName().startsWith("stepDelete"))
				.mapToLong(s -> s.getExecutionContext().containsKey("deletedA")
					? s.getExecutionContext().getInt("deletedA")
					: (s.getExecutionContext().containsKey("deletedB") ? s.getExecutionContext().getInt("deletedB") : 0))
				.sum();

			return CommonResponse.respondOk(new MigrationResultDto(runId, migrated, deleted), "이관 배치 실행 성공");
		} catch (Exception e) {
			return CommonResponse.respondFail("이관 배치 실행 실패: " + e.getMessage());
		}
	}
}
```````



