---
RFC: 17
Status: Proposed
---

# Valkey-Python-Integration-Test-Framework

## Abstract
In this RFC, we are proposing a light weight python testing framework that can be used by Modules and other packages to test against / with the valkey-server. Use cases include Modules in other code packages that need to write integeration tests. Example: ValkeyJSON, Valkey-Bloom

## Motivation

Currently, we can have software developed (e.g. Modules) for Valkey written in different languages - C, C++, Rust. For the purpose of integeration testing against Valkey Servers, without a common framework, each package will need to re-implement the same infrastructure. Having a single Integration Test Framework will solve this problem. A second motivation is simplicity and maintainability. Currently, tests in the valkey core are writtin in tcl files; for new contibutors onboarding, and to allow easy extending the integration test framework (by levegraing existing libraries), using a Language such as Python will help.

## Design Considerations

For minimal requirements, integration will need to support the following:
* ValkeyServer class - Start up / Tear down of a valkey-server
* ValeyClient class - Wrapper for creating a new client & connecting to the server, and supporting common client operations.
* Specifying custom engine binary
* Automatically building the valkey-server binary (pulling from valkey-io project)
* Specifying Custom Startup arguments
* Replication Support
* TLS
* Cluster Mode Enabled / Standalone
* Execution of multiple tests concurrently: Each test running on separate ports

## Interface

1. Example of each test in a test file (class) reusing the same server instance - and having serial / ordered test execution:

[See example file](https://github.com/KarthikSubbarao/valkey-test-framework/blob/unstable/tests/test_reuse_setup_per_class.py)

```
class TestExampleReuseSetupPerClass(ValkeyTestCase):
    """
    Every test will use the same server and client instance and the server will be torn down in the final test.
    This also adds ordering of the tests in a serial manner.
    """

    common_server = None
    common_client = None

    def setup_server_and_client(self):
        server_path = f"{os.path.dirname(os.path.realpath(__file__))}/.build/binaries/{os.environ['SERVER_VERSION']}/valkey-server"
        additional_startup_args = ""
        (
            TestExampleReuseSetupPerClass.common_server,
            TestExampleReuseSetupPerClass.common_client,
        ) = self.create_server(
            testdir=self.testdir,
            server_path=server_path,
            args=additional_startup_args,
            skip_teardown=True,
        )

    @pytest.mark.order(1)
    def test_basic1(self):
        # Set up the server and client just one time.
        self.setup_server_and_client()
        TestExampleReuseSetupPerClass.common_client.execute_command("PING")

    @pytest.mark.order(2)
    def test_basic2(self):
        TestExampleReuseSetupPerClass.common_client.execute_command("PING")
        c = TestExampleReuseSetupPerClass.common_server.get_new_client()
        c.execute_command("PING")

    @pytest.mark.order(3)
    def test_basic3(self):
        TestExampleReuseSetupPerClass.common_client.execute_command("SET K V")
        TestExampleReuseSetupPerClass.common_server.exit()

```

2. Example of each test in a test file (class) having their own server spawned/torn down - and each test in the class has custom server setup.

[See example file](https://github.com/KarthikSubbarao/valkey-test-framework/blob/unstable/tests/test_specific_setup_per_test.py)

```
class TestExampleSpecificSetupPerTest(ValkeyTestCase):
    """
    Every test in this class will have a custom setup that is defined individually.
    """

    def test_per_test_setup(self):
        server_path = f"{os.path.dirname(os.path.realpath(__file__))}/.build/binaries/{os.environ['SERVER_VERSION']}/valkey-server"
        additional_startup_args = ""
        self.server, self.client = self.create_server(
            testdir=self.testdir, server_path=server_path, args=additional_startup_args
        )
        self.client.execute_command("PING")

    def test_per_test_setup_memory_limit_arg_example(self):
        server_path = f"{os.path.dirname(os.path.realpath(__file__))}/.build/binaries/{os.environ['SERVER_VERSION']}/valkey-server"
        additional_startup_args = {"maxmemory": "1kb"}
        self.server, self.client = self.create_server(
            testdir=self.testdir, server_path=server_path, args=additional_startup_args
        )
        self.client.execute_command("PING")
        try:
            self.client.execute_command("SET KEY VAL")
            assert (
                False
            ), f"SET command executed correctly when it should have failed due to memory > 'maxmemory"
        except Exception as e:
            assert str(e) == "command not allowed when used memory > 'maxmemory'."

        self.new_server, self.new_client = self.create_server(
            testdir=self.testdir,
            server_path=server_path,
        )
        self.new_client.execute_command("SET KEY VAL")
```

3. Example of each test in a test file (class) having their own server spawned/torn down - and each test in the class has the same server setup logic.

[See example file](https://github.com/KarthikSubbarao/valkey-test-framework/blob/unstable/tests/test_specific_setup_per_class.py)

```
class ExampleTestCaseBase(ValkeyTestCase):
    @pytest.fixture(autouse=True)
    def setup_test(self, setup):
        server_path = f"{os.path.dirname(os.path.realpath(__file__))}/.build/binaries/{os.environ['SERVER_VERSION']}/valkey-server"
        additional_startup_args = ""
        self.server, self.client = self.create_server(
            testdir=self.testdir, server_path=server_path, args=additional_startup_args
        )


class TestExampleSpecificSetupPerClass(ExampleTestCaseBase):
    """
    Every test will use the same server startup from the ExampleTestCaseBase.
    """

    def test_basic1(self):
        client = self.server.get_new_client()
        client.execute_command("PING")

    def test_basic2(self):
        client = self.server.get_new_client()
        client.execute_command("SET K V")

```

4. Example of each test in a test file (class) reusing the same replication server instances - and having serial / ordered test execution:

[See example file](https://github.com/KarthikSubbarao/valkey-test-framework/blob/unstable/tests/test_reuse_replication_setup_per_class.py)

```
class TestExampleReuseReplicationSetupPerClass(ReplicationTestCase):
    """
    Every test will use the same replication setup and the servers will be torn down in the final test.
    This also adds ordering of the tests in a serial manner.
    """

    common_server = None
    common_client = None
    common_replicas = []

    def setup_server_and_client(self):
        server_path = f"{os.path.dirname(os.path.realpath(__file__))}/.build/binaries/{os.environ['SERVER_VERSION']}/valkey-server"
        # This is just to avoid a delay on startup for setting up replication.
        additional_startup_args = {
            "repl-diskless-sync": "yes",
            "repl-diskless-sync-delay": "0",
        }
        (
            primary_server,
            TestExampleReuseReplicationSetupPerClass.common_client,
        ) = self.create_server(
            testdir=self.testdir,
            server_path=server_path,
            args=additional_startup_args,
            skip_teardown=True,
        )
        TestExampleReuseReplicationSetupPerClass.common_server = primary_server
        TestExampleReuseReplicationSetupPerClass.common_replicas = (
            self.setup_replication(
                num_replicas=1, skip_teardown=True, primary_server=primary_server
            )
        )

    @pytest.mark.order(1)
    def test_replication_basic1(self):
        # Set up the server and client just one time.
        self.setup_server_and_client()
        server = TestExampleReuseReplicationSetupPerClass.common_server
        replicas = TestExampleReuseReplicationSetupPerClass.common_replicas
        client = TestExampleReuseReplicationSetupPerClass.common_client
        server = TestExampleReuseReplicationSetupPerClass.common_server
        replicas = TestExampleReuseReplicationSetupPerClass.common_replicas
        client.execute_command("SET K V")
        self.waitForReplicaToSyncUp(replicas[0])
        assert replicas[0].client.execute_command("GET K") == b"V"

    @pytest.mark.order(2)
    def test_replication_basic2(self):
        server = TestExampleReuseReplicationSetupPerClass.common_server
        replicas = TestExampleReuseReplicationSetupPerClass.common_replicas
        client = TestExampleReuseReplicationSetupPerClass.common_client
        assert replicas[0].client.execute_command("GET K") == b"V"
        client.execute_command("SET K VV")
        self.waitForReplicaToSyncUp(replicas[0])
        assert replicas[0].client.execute_command("GET K") == b"VV"

    @pytest.mark.order(3)
    def test_replication_basic3(self):
        server = TestExampleReuseReplicationSetupPerClass.common_server
        replicas = TestExampleReuseReplicationSetupPerClass.common_replicas
        replicas[0].client.execute_command("CONFIG SET repl-timeout 5") == b"OK"
        assert replicas[0].client.execute_command("GET K") == b"VV"
        server.exit()
        replicas[0].exit()
```

5. Example of each test in a test file (class) having their own replication servers spawned/torn down - and each test in the class has custom server & replication setup.

[See example file](https://github.com/KarthikSubbarao/valkey-test-framework/blob/unstable/tests/test_specific_replication_setup_per_class.py)

```
class TestExampleReplication(ReplicationTestCase):
    @pytest.fixture(autouse=True)
    def setup_test(self, setup):
        # This is just to avoid a delay on startup for setting up replication.
        additional_startup_args = {
            "repl-diskless-sync": "yes",
            "repl-diskless-sync-delay": "0",
        }
        server_path = f"{os.path.dirname(os.path.realpath(__file__))}/.build/binaries/{os.environ['SERVER_VERSION']}/valkey-server"
        self.server, self.client = self.create_server(
            testdir=self.testdir, server_path=server_path, args=additional_startup_args
        )
        self.setup_replication(num_replicas=1)

    def test_replication1(self):
        self.replicas[0].client.execute_command("CONFIG SET repl-timeout 5") == b"OK"
        self.client.execute_command("SET K V")
        self.waitForReplicaToSyncUp(self.replicas[0])
        assert self.replicas[0].client.execute_command("GET K") == b"V"

    def test_replication2(self):
        self.client.execute_command("SET K VV")
        self.waitForReplicaToSyncUp(self.replicas[0])
        assert self.replicas[0].client.execute_command("GET K") == b"VV"
```

## Reference:
https://github.com/valkey-io/valkey-bloom/tree/unstable/tests/valkeytests
