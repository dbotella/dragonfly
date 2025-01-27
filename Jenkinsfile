pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timeout(time: 1, unit: 'HOURS')
    }

    environment {
        SCCACHE_CACHE_SIZE = '6G'
        SCCACHE_ERROR_LOG = '/tmp/sccache_log.txt'
    }

    stages {
        stage('Pre-commit checks') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install pre-commit
                    pre-commit run --show-diff-on-failure --color=always --all-files
                '''
            }
        }

        stage('Build and Test') {
            steps {
                buildAndTest('Debug', 'g++', 'gcc', '-Werror', false)
            }
        }
    }

    post {
        failure {
            archiveArtifacts artifacts: '/tmp/*INFO*, /tmp/failed/*', allowEmptyArchive: true
        }
    }
}

def buildAndTest(buildType, cxxCompiler, cCompiler, cxxFlags, withSanitizers) {
    sh """
        mkdir -p build
        
        # Free up space
        rm -rf /usr/share/dotnet
        rm -rf /usr/local/share/boost
        rm -rf /usr/local/lib/android
        rm -rf /opt/ghc

        # Configure CMake
        ASAN="OFF"
        USAN="OFF"
        if [ "${withSanitizers}" = "true" ]; then
            ASAN="ON"
            USAN="ON"
        fi

        cmake -B build \\
            -DCMAKE_BUILD_TYPE=${buildType} \\
            -GNinja \\
            -DCMAKE_C_COMPILER=${cCompiler} \\
            -DCMAKE_CXX_COMPILER=${cxxCompiler} \\
            -DCMAKE_CXX_FLAGS="${cxxFlags} -no-pie" \\
            -DWITH_AWS:BOOL=OFF \\
            -DWITH_ASAN="\${ASAN}" \\
            -DWITH_USAN="\${USAN}"

        # Build
        cd build
        ninja search_family_test
        ninja src/all

        # Run tests
        GLOG_alsologtostderr=1 GLOG_vmodule=rdb_load=1,rdb_save=1,snapshot=1 \\
        FLAGS_fiber_safety_margin=4096 FLAGS_list_experimental_v2=true \\
        timeout 20m ctest -V -L DFLY

        # Run additional test configurations
        FLAGS_fiber_safety_margin=4096 FLAGS_force_epoll=true \\
        GLOG_vmodule=rdb_load=1,rdb_save=1,snapshot=1 \\
        timeout 20m ctest -V -L DFLY

        FLAGS_fiber_safety_margin=4096 FLAGS_cluster_mode=emulated \\
        timeout 20m ctest -V -L DFLY

        FLAGS_fiber_safety_margin=4096 FLAGS_cluster_mode=emulated \\
        FLAGS_lock_on_hashtags=true timeout 20m ctest -V -L DFLY

        timeout 5m ./dragonfly_test
        timeout 5m ./json_family_test --jsonpathv2=false
        timeout 5m ./tiered_storage_test --vmodule=db_slice=2 --logtostderr
    """
} 