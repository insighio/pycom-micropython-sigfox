def buildVersion
def boards_to_build = ["WiPy", "LoPy", "SiPy", "GPy", "FiPy", "LoPy4"]
def boards_to_test = ["Pycom_Expansion3_Py00ec5f", "Pycom_Expansion3_Py9f8bf5"]
String remote_node = "RPI3" // Need to update functions below to use this variable (which failed in the past)

node {
	PYCOM_VERSION=get_version()
	GIT_TAG = sh (script: 'git rev-parse --short HEAD', returnStdout: true).trim()

    // get pycom-esp-idf source
    stage('Checkout') {
        checkout scm
        sh 'rm -rf esp-idf'
        sh 'git clone --depth=1 --recursive -b master https://github.com/pycom/pycom-esp-idf.git esp-idf'
    }

    stage('mpy-cross') {
        // build the cross compiler first
        sh 'git tag -fa v1.8.6-849-' + GIT_TAG + ' -m \\"v1.8.6-849-' + GIT_TAG + '''\\";
          cd mpy-cross;
          make clean;
          make all'''
    }

    stage('IDF-LIBS') {
        // build the libs from esp-idf
       sh '''export PATH=$PATH:/opt/xtensa-esp32-elf/bin;
        		 export IDF_PATH=${WORKSPACE}/esp-idf;
        		 cd $IDF_PATH/examples/wifi/scan;
        		 make clean && make all'''
    }

 	for (board in boards_to_build) {
		stage(board) {
			def parallelSteps = [:]
            def board_u = board.toUpperCase()
            if (board_u == "LOPY" || board_u == "FIPY"  || board_u == "LOPY4") {
        			parallelSteps[board+"_868"] = boardBuild(board+"_868")
        			parallelSteps[board+"_915"] = boardBuild(board+"_915")
        		}
    			else{
        			parallelSteps[board] = boardBuild(board)
        		}
        		echo 'ParallelSteps: ' + parallelSteps
        		parallel parallelSteps
  		}
  	}

    stash includes: '**/*.bin', name: 'binary'
    stash includes: 'tests/**', name: 'tests'
    stash includes: 'esp-idf/components/esptool_py/**', name: 'esp-idfTools'
    stash includes: 'tools/**', name: 'tools'
    stash includes: 'esp32/tools/**', name: 'esp32Tools'
}

stage ('Flash') {
	def parallelFlash = [:]
	for (board in boards_to_test) {
		parallelFlash[board] = flashBuild(board)
	}
	echo 'ParallelFlash: ' + parallelFlash
	parallel parallelFlash
}

stage ('Test'){
	def parallelTests = [:]
	for (board in boards_to_test) {
		parallelTests[board] = testBuild(board)
	}
	parallel parallelTests
}


def boardBuild(name) {
    def name_u = name.toUpperCase()
    def name_short = name_u.split('_')[0]
    def lora_band = ""
    if (name_u == "LOPY_868" || name_u == "FIPY_868"  || name_u == "LOPY4_868") {
        lora_band = " LORA_BAND=USE_BAND_868"
    }
    else if (name_u == "LOPY_915" || name_u == "FIPY_915" || name_u == "LOPY4_915") {
        lora_band = " LORA_BAND=USE_BAND_915"
    }
    def app_bin = name.toLowerCase() + '.bin'
    return {
    		echo 'version: ' + PYCOM_VERSION
    		release_dir = "${JENKINS_HOME}/release/${JOB_NAME}/" + PYCOM_VERSION + "/" + GIT_TAG + "/"
    		echo 'git_tag: ' + GIT_TAG
    		echo 'release_dir: ' + release_dir
        sh '''export PATH=$PATH:/opt/xtensa-esp32-elf/bin;
        export IDF_PATH=${WORKSPACE}/esp-idf;
        cd esp32;
        make clean BOARD=''' + name_short + lora_band

        sh '''export PATH=$PATH:/opt/xtensa-esp32-elf/bin;
        export IDF_PATH=${WORKSPACE}/esp-idf;
        cd esp32;
        make TARGET=boot -j2 BOARD=''' + name_short + lora_band

        sh '''export PATH=$PATH:/opt/xtensa-esp32-elf/bin;
        export IDF_PATH=${WORKSPACE}/esp-idf;
        cd esp32;
        make TARGET=app -j2 BOARD=''' + name_short + lora_band

        sh '''cd esp32/build/'''+ name_u +'''/release;
        mkdir -p firmware_package;
        mkdir -p '''+ release_dir + ''';
        cd firmware_package;
        cp ../bootloader/bootloader.bin .;
        mv ../application.elf ''' + release_dir + name + "-" + PYCOM_VERSION + '''-application.elf;
        cp ../appimg.bin .;
        cp ../lib/partitions.bin .;
        cp ../../../../boards/''' + name_short + '''/''' + name_u + '''/script .;
        cp ../''' + app_bin + ''' .;
        tar -cvzf ''' + release_dir + name + "-" + PYCOM_VERSION + '''.tar.gz  appimg.bin  bootloader.bin   partitions.bin   script ''' + app_bin
    }
}

def flashBuild(short_name) {
  return {
    String device_name = get_device_name(short_name)
    String board_name_u = get_firmware_name(short_name)
    node("UDOO") {
    	  echo 'Flashing ' + board_name_u + ' on ' + device_name
      sh 'rm -rf *'
      unstash 'binary'
      unstash 'esp-idfTools'
      unstash 'esp32Tools'
      unstash 'tests'
      unstash 'tools'
      sh 'python esp32/tools/pypic.py --port ' + device_name +' --enter'
      sh 'esp-idf/components/esptool_py/esptool/esptool.py --chip esp32 --port ' + device_name +' --baud 921600 erase_flash'
      sh 'python esp32/tools/pypic.py --port ' + device_name +' --enter'
      sh 'esp-idf/components/esptool_py/esptool/esptool.py --chip esp32 --port ' + device_name +' --baud 921600 --before no_reset --after no_reset write_flash -z --flash_mode dio --flash_freq 80m --flash_size detect 0x1000 esp32/build/'+ board_name_u +'/release/bootloader/bootloader.bin 0x8000 esp32/build/'+ board_name_u +'/release/lib/partitions.bin 0x10000 esp32/build/'+ board_name_u +'/release/appimg.bin'
      sh 'python esp32/tools/pypic.py --port ' + device_name +' --exit'
    }
  }
}

def testBuild(board_name) {
  return {
    String device_name = get_device_name(board_name)
    String board_name_u = get_firmware_name(short_name)
	node("UDOO") {
 	  echo 'Testing ' + board_name_u + ' on /dev/' + device_name
		sleep(5) //Delay to skip all bootlog
		dir('tests') {
			timeout(30) {
            		sh './run-tests --target=esp32-' + board_name_u + ' --device ' + device_name
        		}
        	}
        	sh 'python esp32/tools/pypic.py --port ' + device_name +' --enter'
        	sh 'python esp32/tools/pypic.py --port ' + device_name +' --exit'
	}
  }
}

def get_version() {
    def matcher = readFile('esp32/pycom_version.h') =~ 'SW_VERSION_NUMBER (.+)'
    matcher ? matcher[0][1].trim().replace('"','') : null
}

def get_firmware_name(short_name) {
  node {
	def node_info = sh (script: 'cat ${JENKINS_HOME}/pycom-ic.conf || exit 0', returnStdout: true).trim()
	def matcher = node_info =~ short_name + ':(.+)'
    matcher ? matcher[0][1] : "WIPY"
  }
}

def get_device_name(short_name) {
    return "/dev/serial/by-id/usb-" +  short_name + "-if00"   
}

