# ---------------------------------
# Repo-specific variables
# ---------------------------------

IMAGE ?= us-west1-docker.pkg.dev/plainsightai-prod/oci/filter-optical-character-recognition

REPO_NAME ?= filter-optical-character-recognition
REPO_NAME_SNAKECASE ?= filter_optical_character_recognition
REPO_NAME_PASCALCASE ?= FilterOpticalCharacterRecognition

# Generate a timestamp in milliseconds
TIMESTAMP_MS := $(shell date +%s%3N)
OUTPUT_VIDEO := ./output/final_output_$(TIMESTAMP_MS).mp4


# ---------------------------------
# Local pipeline definition
# ---------------------------------
PIPELINE := \
  - VideoIn \
      --sources 'file://video.mp4!loop' \
  - filter_license_plate_object_detection.filter.FilterLicensePlateObjectDetection \
      --mq_log pretty \
      $(if $(FILTER_DEBUG), --debug_mode $(FILTER_DEBUG)) \
      --model_path ./model.pth \
      --output_json_path ./output/license_plate_results.json \
      $(if $(FILTER_WRITE_DETECTIONS_TO_JSON), --write_detections_to_json $(FILTER_WRITE_DETECTIONS_TO_JSON)) \
      --forward_detection_rois true \
      --roi_output_label polygons \
  - crop_filter.filter.CropFilter \
      --mq_log pretty \
      $(if $(FILTER_CROP_FROM_ENV), --crop_from_env $(FILTER_CROP_FROM_ENV)) \
      --mutate_original_frames false \
      --cropped_frame_prefix cropped_ \
  - filter_optical_character_recognition.filter.FilterOpticalCharacterRecognition \
      --mq_log pretty \
      $(if $(FILTER_DEBUG), --debug_mode $(FILTER_DEBUG)) \
      --ocr_engine easyocr \
      --output_json_path ./output/ocr_results.json \
      --topic_pattern cropped_main \
  - filter_license_annotation_demo.filter.FilterLicenseAnnotationDemo \
      --mq_log pretty \
      $(if $(FILTER_DEBUG), --debug_mode $(FILTER_DEBUG)) \
      --cropped_topic_suffix cropped_main \
      --font_scale 4.0 \
      --font_thickness 4 \
      --inset_size 200x70 \
      --inset_margin 10x10 \
  - Webvis

# ---------------------------------
# Targets
# ---------------------------------

.PHONY: download-local-wheels
download-local-wheels:
	@echo "Downloading wheels listed in requirements.local.txt from GAR..."
	@mkdir -p filters
	@ACCESS_TOKEN=$$(gcloud auth print-access-token); \
	while read -r line; do \
		pkgfile=$$(basename $$line); \
		name_version=$$(echo $$pkgfile | sed 's/-py3.*\.whl//' | sed 's/_/-/g'); \
		name=$$(echo $$name_version | sed -E 's/(.*)-[0-9]+\..*/\1/'); \
		version=$$(echo $$name_version | sed -E 's/.*-([0-9]+\..*)/\1/'); \
		echo "→ $$name==$${version}"; \
		pip download --no-deps --dest=filters \
		  --index-url "https://oauth2accesstoken:$$ACCESS_TOKEN@us-west1-python.pkg.dev/plainsightai-prod/python/simple/" \
		  "$$name==$${version}"; \
	done < requirements.local.txt

.PHONY: run
run: # Open the browser and run the filters
	( sleep 3 && xdg-open http://localhost:8000 >/dev/null 2>&1 & )
	filter_runtime run $(PIPELINE)

.PHONY: install
install:  # Install python dependencies and download the model
	@echo "Installing Python dependencies..."
	pip install wheel
	pip install --no-deps -r requirements.local.txt
	pip install . --no-build-isolation

	@if [ ! -f model.pth ]; then \
		echo "model not found, downloading model archive..."; \
		curl -L -o model.zip https://models.openfilter.io/license_plate_detection_model/v0.1.0.zip; \
		echo "Unzipping model archive to current directory..."; \
		unzip -o model.zip; \
		rm model.zip; \
	else \
		echo "model already exists, skipping download."; \
	fi


# ---------------------------------
# Shared makefile include
# ---------------------------------

include build-include/filter.mk
