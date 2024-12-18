# Data-Labeling-
from typing import List
import PIL.Image

import torchvision.models

from cvat_sdk import make_client
import cvat_sdk.models as models
import cvat_sdk.auto_annotation as cvataa

class TorchvisionDetectionFunction:
    def __init__(self, model_name: str, weights_name: str, **kwargs) -> None:
        # load the ML model
        weights_enum = torchvision.models.get_model_weights(model_name)
        self._weights = weights_enum[weights_name]
        self._transforms = self._weights.transforms()
        self._model = torchvision.models.get_model(model_name, weights=self._weights, **kwargs)
        self._model.eval()

    @property
    def spec(self) -> cvataa.DetectionFunctionSpec:
        # describe the annotations
        return cvataa.DetectionFunctionSpec(
            labels=[
                cvataa.label_spec(cat, i)
                for i, cat in enumerate(self._weights.meta['categories'])
            ]
        )

    def detect(
        self, context: cvataa.DetectionFunctionContext, image: PIL.Image.Image
    ) -> list[models.LabeledShapeRequest]:
        # determine the threshold for filtering results
        conf_threshold = context.conf_threshold or 0

        # convert the input into a form the model can understand
        transformed_image = [self._transforms(image)]

        # run the ML model
        results = self._model(transformed_image)

        # convert the results into a form CVAT can understand
        return [
            cvataa.rectangle(label.item(), [x.item() for x in box])
            for result in results
            for box, label, score in zip(result["boxes"], result["labels"], result["scores"])
            if score >= conf_threshold
        ]

# log into the CVAT server
with make_client(host="localhost", credentials=("user", "password")) as client:
    # annotate task 12345 using Faster R-CNN
    cvataa.annotate_task(client, 41617,
        TorchvisionDetectionFunction("fasterrcnn_resnet50_fpn_v2", "DEFAULT", box_score_thresh=0.5),
    )
