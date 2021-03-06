KONG_VERSION ?= 2.8.1
KONG_EE_VERSION ?= ${KONG_VERSION}.1

destroy-kong: kong-destroy
kong-destroy:
	@echo cleaning up previous state
	@-docker rm -f how-to-kong-gateway > /dev/null 2>&1
	@-docker rm -f how-to-kong-database > /dev/null 2>&1
	@-docker network rm how-to-kong-net > /dev/null 2>&1

kong-init:
	@-docker network create how-to-kong-net > /dev/null 2>&1

wait-for-db:
	@echo waiting for database
	@while ! docker exec how-to-kong-database pg_isready >/dev/null 2>&1; do echo waiting for database; sleep 1; done
	@echo database is ready

wait-for-kong:
	@echo waiting for kong
	@while ! docker exec how-to-kong-gateway kong health >/dev/null 2>&1; do echo waiting for kong; sleep 2; done
	@echo kong is ready

kong-db: kong-init
	@echo starting database
	@docker run -d --name how-to-kong-database --network=how-to-kong-net -p 5432:5432 -e "POSTGRES_USER=kong" -e "POSTGRES_DB=kong" -e "POSTGRES_PASSWORD=kong" postgres:9.6 > /dev/null 2>&1
	@-make wait-for-db
	@sleep 1 # not sure why but this 1 second seems required to allow the socket to fully open and db to be ready

kong: kong-oss
kong-ce: kong-oss
kong-oss: kong-destroy kong-db
	@echo bootstrapping database
	@docker run --rm --network=how-to-kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=how-to-kong-database" -e "KONG_PG_USER=kong" -e "KONG_PG_PASSWORD=kong" -e "KONG_CASSANDRA_CONTACT_POINTS=how-to-kong-database" kong:latest kong migrations bootstrap > /dev/null 2>&1
	@docker run -d --name how-to-kong-gateway --network=how-to-kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=how-to-kong-database" -e "KONG_PG_USER=kong" -e "KONG_PG_PASSWORD=kong" -e "KONG_CASSANDRA_CONTACT_POINTS=how-to-kong-database" -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" -e "KONG_PROXY_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" -p 8000:8000 -p 8443:8443 -p 127.0.0.1:8001:8001 -p 127.0.0.1:8444:8444 kong:${KONG_VERSION} > /dev/null 2>&1
	@make wait-for-kong
	@make mock-service
	@make validate-kong

kong-free: kong-destroy kong-db
	@echo bootstrapping database
	@docker run --rm --network=how-to-kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=how-to-kong-database" -e "KONG_PG_PASSWORD=kong" -e "KONG_PASSWORD=test" kong/kong-gateway:2.8.1.1-alpine kong migrations bootstrap
	@docker run -d --name how-to-kong-gateway --network=how-to-kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=how-to-kong-database" -e "KONG_PG_USER=kong" -e "KONG_PG_PASSWORD=kong" -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" -e "KONG_PROXY_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" -e "KONG_ADMIN_GUI_URL=http://localhost:8002" -p 8000:8000 -p 8443:8443 -p 8001:8001 -p 8444:8444 -p 8002:8002 -p 8445:8445 -p 8003:8003 -p 8004:8004 kong/kong-gateway:${KONG_EE_VERSION}-alpine
	@make wait-for-kong
	@make mock-service
	@make validate-kong

kong-ee: kong-destroy kong-db
ifndef KONG_LICENSE_FILE
$(error KONG_LICENSE_FILE is not set)
endif
	@echo bootstrapping database
	@docker run --rm --network=how-to-kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=how-to-kong-database" -e "KONG_PG_PASSWORD=kong" -e "KONG_PASSWORD=test" kong/kong-gateway:2.8.1.1-alpine kong migrations bootstrap
	@KONG_LICENSE_DATA='$(shell cat ${KONG_LICENSE_FILE})' docker run -d --name how-to-kong-gateway --network=how-to-kong-net -e "KONG_DATABASE=postgres" -e "KONG_PG_HOST=how-to-kong-database" -e "KONG_PG_USER=kong" -e "KONG_PG_PASSWORD=kong" -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" -e "KONG_PROXY_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" -e "KONG_ADMIN_GUI_URL=http://localhost:8002" -e KONG_LICENSE_DATA -p 8000:8000 -p 8443:8443 -p 8001:8001 -p 8444:8444 -p 8002:8002 -p 8445:8445 -p 8003:8003 -p 8004:8004 kong/kong-gateway:${KONG_EE_VERSION}-alpine 
	@make wait-for-kong
	@make mock-service
	@make validate-kong

mock-service: 
	@echo 'adding mock service at path /mock'
	@curl -i -X POST http://localhost:8001/services --data name=mock --data url='http://mockbin.org' > /dev/null 2>&1
	@curl -i -X POST http://localhost:8001/services/mock/routes --data 'paths[]=/mock' --data name=mocking > /dev/null 2>&1

validate-kong:
	@curl -i http://localhost:8001 >/dev/null 2>&1 && echo "kong is up" || echo "issues starting kong"
