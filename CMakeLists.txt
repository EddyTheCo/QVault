cmake_minimum_required(VERSION 3.24)
option(USE_QML "Create QML types and modules" ON)
include(${CMAKE_CURRENT_BINARY_DIR}/local_conf.cmake OPTIONAL)
include(InstallRequiredSystemLibraries)
include(FetchContent)
FetchContent_Declare(
	ccommon
	GIT_REPOSITORY https://github.com/EddyTheCo/Common.git
	GIT_TAG v0.4.0
	)
FetchContent_MakeAvailable(ccommon)
version_from_git(
	LOG       OFF
	TIMESTAMP "%Y%m%d%H%M%S"
	)
project(QtVault VERSION ${VERSION} DESCRIPTION "Store encrypted data on the local system" LANGUAGES CXX)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
include(GNUInstallDirs)

if(USE_QML)
	find_package(Qt6 COMPONENTS Core Gui Quick Core Network)
else()
	find_package(Qt6 COMPONENTS Core Network)
endif()


find_package(OpenSSL)


if(NOT OPENSSL_FOUND)
	if(EMSCRIPTEN OR ANDROID)
		if(EMSCRIPTEN)
			set(builturl "https://github.com/EddyTheCo/install-openssl-action/releases/latest/download/openssl-wasm_true-android_none.tar")
		endif(EMSCRIPTEN)
		if(ANDROID)
			set(android_abi_ossl "android-arm")
			if(ANDROID_ABI STREQUAL "arm64-v8a")
				set(android_abi_ossl "android-arm64")
			endif()
			set(builturl "https://github.com/EddyTheCo/install-openssl-action/releases/latest/download/openssl-wasm_false-android_${android_abi_ossl}.tar")
		endif(ANDROID)
		FetchContent_Declare(
			openssl
			DOWNLOAD_EXTRACT_TIMESTAMP true
			URL  ${builturl}
			)
		FetchContent_GetProperties(openssl)
		if(NOT openssl_POPULATED)
			message(STATUS "openssl prebuilt will be downloaded from ${builturl} if not found on the system")
			FetchContent_Populate(openssl)
			message(STATUS "openssl is installed in ${openssl_SOURCE_DIR}")
			set(OPENSSL_FOUND ON)
		endif(NOT openssl_POPULATED)
		include(${openssl_SOURCE_DIR}/openssl.cmake)
		create_targets()
	endif(EMSCRIPTEN OR ANDROID )
endif(NOT OPENSSL_FOUND)

if(OPENSSL_FOUND AND TARGET OpenSSL::Crypto AND Qt6_FOUND)
	qt_standard_project_setup()
	add_library(qvault include/vault.hpp src/vault.cpp)
	add_library(${PROJECT_NAME}::qvault ALIAS qvault)
	set_target_properties(qvault PROPERTIES VERSION ${VERSION} SOVERSION ${VERSION_MAJOR})
	if(USE_QML)
		qt6_add_qml_module(qvault
			URI  Esterv.Utils.Vault
			VERSION 1.0
			RESOURCE_PREFIX
			"/esterVtech.com/imports"
			OUTPUT_TARGETS out_targets_var
			OUTPUT_DIRECTORY
			${CMAKE_BINARY_DIR}/Esterv/Utils/Vault
			IMPORT_PATH ${CMAKE_BINARY_DIR}
			)
		add_library(${PROJECT_NAME}::qvaultplugin ALIAS qvaultplugin)
		target_link_libraries(qvault PUBLIC Qt6::Gui Qt6::Quick)
		target_compile_definitions(qvault PRIVATE USE_QML)

		install(TARGETS ${out_targets_var}
			EXPORT ${PROJECT_NAME}-config
			DESTINATION ${CMAKE_INSTALL_LIBDIR}
			COMPONENT vault
			RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT vault
			)
		install(DIRECTORY ${CMAKE_BINARY_DIR}/Esterv/Utils/Vault
			DESTINATION ${CMAKE_INSTALL_LIBDIR}/Esterv/Utils
			COMPONENT vault
			)
		install(TARGETS qvaultplugin
			EXPORT ${PROJECT_NAME}-config
			DESTINATION ${CMAKE_INSTALL_LIBDIR}/Esterv/Utils/Vault
			COMPONENT vault
			RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT vault
			)
	endif(USE_QML)


	target_include_directories(qvault PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
		"$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/Esterv/>")

	target_link_libraries(qvault PUBLIC Qt6::Core Qt6::Network OpenSSL::Crypto)
	target_compile_definitions(qvault PRIVATE WINDOWS_QVAULT)
	if(EMSCRIPTEN)
		target_compile_definitions(qvault PRIVATE USE_EMSCRIPTEN)
	endif(EMSCRIPTEN)

	if(NOT EMSCRIPTEN AND NOT ANDROID)
		add_executable(QVaultCli src/QVaultCli.cpp)
		target_link_libraries(QVaultCli PRIVATE qvault)
		add_executable(${PROJECT_NAME}::QVaultCli ALIAS QVaultCli)
		set_target_properties(QVaultCli PROPERTIES
			WIN32_EXECUTABLE ON
			MACOSX_BUNDLE ON
			)
		install(TARGETS QVaultCli COMPONENT vaultCli
			BUNDLE DESTINATION . COMPONENT vaultCli
			)
		if(QTDEPLOY)
			qt_generate_deploy_app_script(
				TARGET QVaultCli
				OUTPUT_SCRIPT deploy_script
				)
			install(SCRIPT ${deploy_script} COMPONENT vaultCli)
		endif(QTDEPLOY)
	endif(NOT EMSCRIPTEN AND NOT ANDROID)


	install(TARGETS qvault
		EXPORT ${PROJECT_NAME}-config
		DESTINATION ${CMAKE_INSTALL_LIBDIR}
		COMPONENT vault
		RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT vault
		)
	install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/
		DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/Esterv
		COMPONENT ${PROJECT_NAME}
		)
	build_exports()
	if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
		add_subdirectory(examples)
		build_cpack()
	endif(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)

endif(OPENSSL_FOUND AND TARGET OpenSSL::Crypto AND Qt6_FOUND)

if(BUILD_DOCS)
	build_docs()
endif()
