# DB 마이그레이션
 
`postgres-schema.sql`(최초 스키마)이 적용된 뒤, 여기 있는 파일들을 **번호 순서대로,
한 번 적용한 파일은 절대 수정하지 않고** 실행한다. 뭔가 더 바꿔야 하면 새 번호로
새 파일을 추가한다.
 
## 처음 설정할 때 (한 번만)
 
```bash
docker exec -i <컨테이너명> psql -U dlp -d dlp -f migrations/000_create_schema_migrations.sql
```
 
## 이후 각 마이그레이션
 
```bash
docker exec -i <컨테이너명> psql -U dlp -d dlp -f migrations/001_add_ner_entity_types.sql
```
 
## 지금 뭐가 적용됐는지 확인
 
```bash
docker exec -i <컨테이너명> psql -U dlp -d dlp -c "SELECT * FROM schema_migrations ORDER BY filename;"
```
 
## 규칙
 
- 모든 마이그레이션 SQL은 **idempotent**해야 한다 (`ON CONFLICT ... DO NOTHING` 등) —
  실수로 두 번 실행해도 에러 없이 안전해야 한다.
- 파일 끝에 반드시 자기 자신을 `schema_migrations`에 기록하는 INSERT를 넣는다.
- 파일명은 `NNN_설명.sql` 형식, 번호는 3자리, 항상 증가.
- 한 번 커밋된 파일은 수정 금지. 잘못됐으면 그걸 고치는 새 마이그레이션을 추가한다.
지금은 각자 수동으로 `docker exec ... -f`를 실행하는 방식이다. 마이그레이션이
늘어나서 수동 실행이 부담되면 그때 "미적용 파일만 순서대로 실행하는 스크립트"를
추가하는 걸 고려한다 — 지금 단계에서는 과하다.