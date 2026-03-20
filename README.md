# Превью кода из приложения Clooqs для РЭД (могла бы быть отсылка к фильму РЭД, но её не будет)


## Cubit, отвечающий за управление состояния приложения. Вызывает методы из Repository, который в свою очередь общается по API
```dart
import 'dart:io';

import 'package:clooqs/core/models/comment/comment_model.dart';
import 'package:clooqs/core/models/fashion/fashion_model.dart';
import 'package:clooqs/core/models/feed/feed_model.dart';
import 'package:clooqs/core/models/tasks/task_model.dart';
import 'package:clooqs/core/models/user/user_model.dart';
import 'package:clooqs/core/services/fashion_cleanup_service.dart';
import 'package:clooqs/core/api/endpoints.dart';
import 'package:clooqs/features/home/presentation/views/fashion/data/fashion_repository.dart';
import 'package:clooqs/core/mixins/fashion_block_mixin.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

part 'fashion_state.dart';

class FashionCubit extends Cubit<FashionState> with FashionBlockMixin {
  FashionCubit() : super(FashionInitial()) {
    _initializeBlockingSystem();
  }

  /// Инициализирует систему блокировки пользователей
  Future<void> _initializeBlockingSystem() async {
    try {
      initializeBlockService();
    } catch (e) {
      print('Ошибка при инициализации системы блокировки в FashionCubit: $e');
    }
  }

  FashionRepository fashionRepository = FashionRepository();

  resetState() async {
    emit(FashionInitial());
  }

  Future<FashionModel?> getFashion({int? id, bool loading = true}) async {
    try {
      if (kDebugMode) {
        print('🔗 FashionCubit: Starting to load fashion with ID: $id');
      }
      if (loading) {
        emit(FashionLoading());
      }
      FashionModel fashion = await fashionRepository.getFashion(id: id);
      if (kDebugMode) {
        final description = fashion.description;
        final preview = description == null
            ? 'No description'
            : (description.length > 30
                ? description.substring(0, 30)
                : description);
        print(
            '✅ FashionCubit: Successfully loaded fashion: ${fashion.id} - $preview...');
      }
      emit(FashionSuccess(fashion: fashion));
      return fashion;
    } catch (e) {
      if (kDebugMode) {
        print('❌ FashionCubit: Error loading fashion with ID $id: $e');
      }
      emit(FashionError(error: e.toString()));
    }
    return null;
  }

  getAllFashions({int? userId}) async {
    try {
      emit(FashionLoading());
      List<FashionModel> fashions = await fashionRepository.getAllFashions(
        userId: userId,
      );

      print('📋 getAllFashions loaded ${fashions.length} fashions');
      if (fashions.isNotEmpty) {
        print(
            '📋 First fashion ID: ${fashions.first.id}, status: ${fashions.first.status}');
      }

      // fashions = fashions
      //     .where((element) =>
      //         element.fashionImages!.isNotEmpty &&
      //         (element.status == null || element.status == 'published'))
      //     .toList();
      emit(AllFashionSuccess(fashions: fashions, userId: userId));
    } catch (e) {
      if (e.toString().contains('UserBlockedException')) {
        // Получаем username пользователя или используем заглушку
        final username = userId?.toString() ?? 'user';
        emit(FashionUserBlockedError(
          username: username,
          message: 'Пользователь заблокирован',
        ));
      } else {
        if (kDebugMode) {}
        emit(FashionError(error: e.toString()));
      }
    }
  }

  getAllFashionsPagination({required String nextPage}) async {
    try {
      emit(FashionLoading());
      FeedResult fashions = await fashionRepository.getAllFashionsPagination(
        nextPage: nextPage,
      );
      // fashions = fashions
      //     .where((element) =>
      //         element.fashionImages!.isNotEmpty &&
      //         (element.status == null || element.status == 'published'))
      //     .toList();
      emit(AllFashionPaginationSuccess(
        fashions: fashions,
      ));
    } catch (e) {
      if (kDebugMode) {}
      emit(FashionError(error: e.toString()));
    }
  }

  createFashion(String? baseGender) async {
    try {
      emit(CreateFashionLoading());
      FashionModel fashion =
          await fashionRepository.createFashion(baseGender: baseGender);
      emit(FashionSuccess(fashion: fashion));
    } catch (e) {
      if (kDebugMode) {}
      emit(FashionError(error: e.toString()));
    }
  }

  Future<bool> deleteDraft({required int id, BuildContext? context}) async {
    try {
      emit(DeleteDraftLoading());
      bool result = await fashionRepository.deleteDraft(id: id);
      if (result) {
        // Очищаем fashion из всех мест в приложении
        if (context != null) {
          FashionCleanupService.removeFashionFromAllPlaces(context, id);
        }

        // Удаляем образ из текущего списка (AllFashionSuccess)
        if (state is AllFashionSuccess) {
          final currentState = state as AllFashionSuccess;
          List<FashionModel> updatedFashions =
              List.from(currentState.fashions ?? []);
          updatedFashions.removeWhere((f) => f.id == id);

          emit(AllFashionSuccess(
              fashions: updatedFashions, userId: currentState.userId));
        }
        // Удаляем образ из пагинированного списка (AllFashionPaginationSuccess)
        else if (state is AllFashionPaginationSuccess) {
          final currentState = state as AllFashionPaginationSuccess;
          if (currentState.fashions?.results != null) {
            List<FashionModel> updatedFashions =
                List.from(currentState.fashions!.results);
            updatedFashions.removeWhere((f) => f.id == id);

            // Обновляем результаты в FeedResult
            currentState.fashions!.results = updatedFashions;

            emit(AllFashionPaginationSuccess(fashions: currentState.fashions));
          } else {
            getAllFashions();
          }
        } else {
          getAllFashions();
        }

        // Даем время на обработку предыдущих состояний, затем эмитим DeleteDraftSuccess
        await Future.delayed(const Duration(milliseconds: 100));
        emit(DeleteDraftSuccess());

        return true;
      }
      return false;
    } catch (e) {
      emit(FashionError(error: e.toString()));
      return false;
    }
  }

  deleteFashionImage({required int fashionId, required int photoId}) async {
    try {
      bool deleted =
          await fashionRepository.deleteFashionImage(fashionId, photoId);
      if (deleted) {
        emit(FashionPhotoDeleted(photoId: photoId));
      }
    } catch (e) {
      emit(FashionError(error: e.toString()));
    }
  }

  deleteProduct({required int fashionId, required int productId}) async {
    try {
      bool deleted = await fashionRepository.deleteProduct(
        fashionId,
        productId,
      );
      if (deleted) {
        emit(ProductDeleted(productId: productId));
      }
    } catch (e) {
      emit(FashionError(error: e.toString()));
    }
  }

  closeBadPhoto() {
    emit(FashionInitial());
  }

  updateFashion({
    required FashionModel? fashionModel,
    required bool disabled,
  }) async {
    if (fashionModel == null) {
      emit(FashionPublicateError(error: 'Образ не найден'));
      return;
    }
    if (disabled || state is CreateFashionLoading) {
      emit(FashionPublicateError(error: 'Образ'));
      return;
    }
    emit(CreateFashionLoading());
    try {
      await updateDescription(fashionModel: fashionModel, creating: true);
      FashionModel fashion = await fashionRepository.publish(
        fashionModel: fashionModel,
      );

      // Обновляем список образов: ставим новый в начало и удаляем дубликаты
      if (state is AllFashionSuccess) {
        final currentState = state as AllFashionSuccess;

        // Создаем глубокие копии всех образов, чтобы избежать мутации общих объектов
        List<FashionModel> updatedFashions = (currentState.fashions ?? [])
            .map((f) => FashionModel.fromJson(f.toJson()))
            .toList();

        // Удаляем дубликаты если образ уже есть в списке
        updatedFashions.removeWhere((f) => f.id == fashion.id);

        // Создаем копию опубликованного образа и добавляем в начало
        updatedFashions.insert(0, FashionModel.fromJson(fashion.toJson()));

        emit(FashionPublicateSuccess(fashion: fashion));
        emit(AllFashionSuccess(
            fashions: updatedFashions, userId: currentState.userId));
      }
      // Обновляем пагинированный список
      else if (state is AllFashionPaginationSuccess) {
        final currentState = state as AllFashionPaginationSuccess;
        if (currentState.fashions?.results != null) {
          // Создаем глубокие копии всех образов
          List<FashionModel> updatedFashions = currentState.fashions!.results
              .map((f) => FashionModel.fromJson(f.toJson()))
              .toList();

          // Удаляем дубликаты если образ уже есть в списке
          updatedFashions.removeWhere((f) => f.id == fashion.id);

          // Создаем копию опубликованного образа и добавляем в начало
          updatedFashions.insert(0, FashionModel.fromJson(fashion.toJson()));

          // Обновляем результаты в FeedResult
          currentState.fashions!.results = updatedFashions;

          emit(FashionPublicateSuccess(fashion: fashion));
          emit(AllFashionPaginationSuccess(fashions: currentState.fashions));
        } else {
          emit(FashionPublicateSuccess(fashion: fashion));
          getAllFashions();
        }
      } else {
        emit(FashionPublicateSuccess(fashion: fashion));
        getAllFashions();
      }
    } catch (e) {
      if (kDebugMode) {}
      emit(FashionPublicateError(error: e.toString()));
    }
  }

  updateDescription(
      {required FashionModel fashionModel, bool creating = false}) async {
    try {
      // emit(CreateFashionLoading());
      FashionModel fashion = await fashionRepository.updateDescription(
        fashionModel: fashionModel,
      );
      if (!creating) {
        emit(FashionDescriptionSuccess(fashion: fashion));
      }
    } catch (e) {
      if (kDebugMode) {}
      emit(FashionDescriptionError(error: e.toString()));
    }
  }

  updateCommentsValue({
    required FashionModel fashionModel,
  }) async {
    try {
      emit(FashionCommentsValueSuccess(fashion: fashionModel));
    } catch (e) {
      if (kDebugMode) {}
    }
  }

  uploadImage({required int fashionId, required File image}) async {
    try {
      emit(FashionImageLoading());

      FashionImage fashion = await fashionRepository.uploadImage(
        fashionId: fashionId,
        file: image,
      );

      // Preload the uploaded image before emitting success
      if (fashion.photo.isNotEmpty) {
        try {
          // Ensure absolute URL for NetworkImage
          final String imageUrl = fashion.photo.startsWith('http')
              ? fashion.photo
              : '${Endpoints.mediaUrl}${fashion.photo.startsWith('/') ? fashion.photo.substring(1) : fashion.photo}';
          final imageProvider = NetworkImage(imageUrl);
          imageProvider.resolve(const ImageConfiguration());
        } catch (e) {
          if (kDebugMode) {
            print('Ошибка прекеширования изображения: $e');
          }
        }
      }

      emit(FashionImageSuccess(fashionImage: fashion));
    } catch (e) {
      if (kDebugMode) {}
      emit(
        FashionImageError(
          error: 'Фото не соответствует условиям'.toString(),
          file: image,
        ),
      );
    }
  }

  addProduct({required String url, required FashionModel fashionModel}) async {
    try {
      emit(AddProductLoading());
      TaskModel taskModel = await fashionRepository.addProduct(
        url: url,
        fashionModel: fashionModel,
      );

      emit(ProductLoading(taskModel: taskModel));
      // emit(AddProductSuccess(product: product));
    } catch (e) {
      if (kDebugMode) {}
      if (e.toString().contains('type')) {
        emit(ProductLoadingError(
            error: 'Попробуйте ещё раз или используйте другую ссылку'));
      }
      emit(ProductLoadingError(error: e.toString()));
      // emit(ProductLoadingError(error: 'Не удалось прикрепить товар'.toString()));
    }
  }

  resetUploadingState() {
    emit(FashionInitial());
  }

  Future<void> cancelProductTask({required String taskId}) async {
    try {
      await fashionRepository.cancelTask(taskId: taskId);
      emit(FashionInitial());
    } catch (e) {
      if (kDebugMode) {
        print('❌ Ошибка отмены задачи: $e');
      }
      emit(FashionInitial());
    }
  }

  checkProductStatus({
    required String taskId,
    required int fashionId,
  }) async {
    if (state is ProductLoadingError) return;
    try {
      TaskModel taskModel =
          await fashionRepository.checkProduct(taskId: taskId);
      if (taskModel.status == TaskStatus.success) {
        if (taskModel.productResult?.photo != null &&
            taskModel.productResult?.name != null &&
            taskModel.productResult?.price != null) {
          // emit(AddProductLongLoading());
          // Future.delayed(const Duration(milliseconds: 5000), () {
          //   emit(AddProductTooLongLoading());
          // });
          // Future.delayed(const Duration(milliseconds: 10000), () {
          //   createFashion();
          //   emit(AddProductSuccess(productResult: taskModel.productResult!));
          // });
          emit(AddProductSuccess(productResult: taskModel.productResult!));
          // createFashion();
        } else {
          emit(
            ProductLoadingError(
              error: 'Не удалось загрузить товар',
              url: taskModel.productResult?.photo,
            ),
          );
        }
      } else if (taskModel.status == TaskStatus.failure) {
        emit(ProductLoadingError(error: 'Не удалось загрузить товар'));
      } else {
        if (state is AddProductLongLoading ||
            state is AddProductTooLongLoading) {
          emit(AddProductTooLongLoading());
          return;
        }
        emit(AddProductLongLoading());
      }
    } catch (e) {
      // if (state is AddProductLongLoading || state is AddProductTooLongLoading) {
      //   emit(AddProductTooLongLoading());
      //   return;
      // }
      // emit(AddProductLongLoading());
      emit(ProductLoadingError(error: e.toString()));
      if (kDebugMode) {}
    }
  }

  likeFashion({required int? fashionId, required int pageId}) async {
    if (fashionId == null) return;
    // Предотвращаем множественные вызовы если уже лайкаем
    if (state is FashionLiked && (state as FashionLiked).fashionId == fashionId)
      return;

    try {
      emit(FashionLiked(fashionId: fashionId, pageId: pageId));
      await fashionRepository.likeFashion(
        fashionId: fashionId,
      );
    } catch (e) {
      if (kDebugMode) {}
      emit(FashionError(error: e.toString()));
    }
  }

  unlikeFashion({required int? fashionId, required int pageId}) async {
    if (fashionId == null) return;
    // Предотвращаем множественные вызовы если уже убираем лайк
    if (state is FashionUnliked &&
        (state as FashionUnliked).fashionId == fashionId) return;

    try {
      emit(FashionUnliked(fashionId: fashionId, pageId: pageId));
      await fashionRepository.unlikeFashion(
        fashionId: fashionId,
      );
    } catch (e) {
      if (kDebugMode) {}
      emit(FashionError(error: e.toString()));
    }
  }

  getFashionLikes({required int? fashionId}) async {
    if (fashionId == null) return;
    emit(GetFashionLikesLoading(fashionId: fashionId));
    try {
      emit(FashionLoading());
      List<UserModel> users = await fashionRepository.getFashionLikes(
        fashionId: fashionId,
      );
      emit(GetFashionLikes(fashionId: fashionId, users: users));
    } catch (e) {
      if (kDebugMode) {}
      emit(GetFashionLikesError(fashionId: fashionId, error: e.toString()));
    }
  }

  getComments({required int? fashionId}) async {
    if (fashionId == null) return;
    emit(GetFashionCommentsLoading(fashionId: fashionId));
    try {
      List<CommentModel> comments =
          await fashionRepository.getComments(fashionId: fashionId);
      emit(GetFashionComments(fashionId: fashionId, comments: comments));
    } catch (e) {
      emit(GetFashionCommentsError(fashionId: fashionId, error: e.toString()));
    }
  }

  sendComment({required int? fashionId, required String text}) async {
    if (fashionId == null) return;
    if (state is AddCommentLoading) return;

    try {
      emit(AddCommentLoading(fashionId: fashionId));
      CommentModel comment = await fashionRepository.sendComment(
        fashionId: fashionId,
        text: text,
      );
      emit(AddCommentSuccess(fashionId: fashionId, comment: comment));
    } catch (e) {
      if (kDebugMode) {}
      emit(AddCommentError(fashionId: fashionId, error: e.toString()));
    }
  }

  deleteComment(
      {required int? fashionId, required int commentId, int? parentId}) async {
    if (fashionId == null) return;
    try {
      // emit(DeleteCommentLoading(fashionId: fashionId));
      bool result = await fashionRepository.deleteComment(
        fashionId: fashionId,
        commentId: commentId,
        parentId: parentId,
      );
      if (result) {
        emit(DeleteCommentSuccess(fashionId: fashionId, commentId: commentId));
      }
    } catch (e) {
      if (kDebugMode) {}
      emit(DeleteCommentError(fashionId: fashionId, error: e.toString()));
    }
  }

  addAnswerToComment(
      {required int? fashionId,
      required int commentId,
      required String text}) async {
    if (fashionId == null) return;
    try {
      emit(
        AddAnswerToCommentLoading(fashionId: fashionId, commentId: commentId),
      );
      CommentModel comment = await fashionRepository.addAnswerToComment(
        fashionId: fashionId,
        commentId: commentId,
        text: text,
      );
      emit(
        AddAnswerToCommentSuccess(
          fashionId: fashionId,
          comment: comment,
          commentId: commentId,
        ),
      );
    } catch (e) {
      if (kDebugMode) {}
      emit(AddAnswerToCommentError(fashionId: fashionId, error: e.toString()));
    }
  }

  getAnswersToComment({required int? fashionId, required int parentId}) async {
    if (fashionId == null) return;
    try {
      emit(
          GetAnswersToCommentLoading(fashionId: fashionId, parentId: parentId));
      List<CommentModel> comments = await fashionRepository.getAnswersToComment(
        fashionId: fashionId,
        parentId: parentId,
      );
      emit(GetAnswersToCommentSuccess(
        fashionId: fashionId,
        comments: comments,
        parentId: parentId,
      ));
    } catch (e) {
      if (kDebugMode) {}
      emit(GetAnswersToCommentError(fashionId: fashionId, error: e.toString()));
    }
  }

  addAnswerToAnswer({
    required int? fashionId,
    required int parentId,
    required int commentId,
    required String text,
  }) async {
    if (fashionId == null) return;
    try {
      emit(
        AddAnswerToCommentLoading(fashionId: fashionId, commentId: commentId),
      );
      CommentModel comment = await fashionRepository.addAnswerToAnswer(
        fashionId: fashionId,
        commentId: commentId,
        parentId: parentId,
        text: text,
      );
      emit(
        AddAnswerToCommentSuccess(
          fashionId: fashionId,
          comment: comment,
          commentId: parentId,
        ),
      );
    } catch (e) {
      if (kDebugMode) {}
      emit(AddAnswerToCommentError(fashionId: fashionId, error: e.toString()));
    }
  }

  editComment({
    required int? fashionId,
    required int commentId,
    required String text,
    int? parentId,
  }) async {
    if (fashionId == null) return;
    try {
      emit(EditCommentLoading(fashionId: fashionId, commentId: commentId));
      CommentModel comment = await fashionRepository.editComment(
        fashionId: fashionId,
        commentId: commentId,
        text: text,
        parentId: parentId,
      );
      emit(EditCommentSuccess(
        fashionId: fashionId,
        comment: comment,
        commentId: commentId,
      ));
    } catch (e) {
      if (kDebugMode) {}
      emit(EditCommentError(fashionId: fashionId, error: e.toString()));
    }
  }

  reportFashion({
    required int fashionId,
    required String reason,
    required String description,
  }) async {
    // if (description.isEmpty) {
    //   emit(ReportError(error: 'Заполните описание жалобы', alert: false));
    //   return;
    // }
    try {
      bool result = await fashionRepository.reportFashion(
        fashionId: fashionId,
        reason: reason,
        description: description,
      );

      if (result) {
        emit(ReportSuccess());
      } else {
        emit(ReportError(error: 'Не удалось пожаловаться'));
      }
    } catch (e) {
      emit(ReportError(error: e.toString()));
    }
  }

  /// Блокирует пользователя и обновляет все списки fashion'ов
  Future<void> blockUserFromFashions(UserModel user) async {
    await user.blockUser();

    // Обновляем текущие состояния, удаляя контент от заблокированного пользователя
    if (state is AllFashionSuccess) {
      final currentState = state as AllFashionSuccess;
      if (currentState.fashions != null) {
        final filteredFashions =
            await filterBlockedFashions(currentState.fashions!);
        emit(AllFashionSuccess(
            fashions: filteredFashions, userId: currentState.userId));
      }
    }

    if (state is AllFashionPaginationSuccess) {
      final currentState = state as AllFashionPaginationSuccess;
      if (currentState.fashions?.results != null) {
        final filteredFashions =
            await filterBlockedFashions(currentState.fashions!.results);
        currentState.fashions!.results = filteredFashions;
        emit(AllFashionPaginationSuccess(fashions: currentState.fashions));
      }
    }
  }

  /// Блокирует владельца fashion'а
  Future<void> blockFashionOwner(FashionModel fashion) async {
    if (fashion.owner != null) {
      await blockUserFromFashions(fashion.owner!);
    }
  }

  /// Разблокирует пользователя
  Future<void> unblockUserFromFashions(UserModel user) async {
    await user.unblockUser();
  }
}
```

## Repository, который взаимодействует с сервером и преобразовывает ответы в модели
```dart
import 'dart:io';

import 'package:clooqs/core/exceptions/user_blocked_exception.dart';
import 'package:clooqs/core/mixins/business_account_mixin.dart';
import 'package:clooqs/core/models/comment/comment_model.dart';
import 'package:clooqs/core/models/fashion/fashion_model.dart';
import 'package:clooqs/core/models/feed/feed_model.dart';
import 'package:clooqs/core/models/tasks/task_model.dart';
import 'package:clooqs/core/models/user/user_model.dart';
import 'package:clooqs/core/service/service_locator.dart';
import 'package:dio/dio.dart';

import 'package:clooqs/core/utils/delete_operation_helper.dart';

class FashionRepository with BusinessAccountMixin {
  final Dio _dio = sl<Dio>();

  Future<FashionModel> getFashion({int? id}) async {
    try {
      String endpoint;
      if (id != null) {
        // GET /api/fashions/{profile_id}/{fashion_id}
        endpoint = await getFashionByIdEndpoint(id);
      } else {
        // GET /api/fashions/{profile_id}
        endpoint = await getFashionProfileEndpoint();
      }

      Response response = await _dio.get(endpoint);

      print(response.data);

      try {
        if (response.statusCode == 200 || response.statusCode == 201) {
          return FashionModel.fromJson(response.data);
        }
      } catch (e) {
        throw e.toString();
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<List<FashionModel>> getAllFashions({int? userId}) async {
    try {
      Map<String, dynamic>? queryParameters;
      if (userId != null) {
        queryParameters = {
          'user_id': userId,
        };
      }
      Response response = await _dio.get(
        await getFashionsEndpoint(),
        queryParameters: queryParameters,
      );

      if (response.statusCode == 200 ||
          response.statusCode == 201 ||
          response.statusCode == 202) {
        try {
          List<FashionModel> fashions = (response.data as List)
              .map((json) => FashionModel.fromJson(json))
              .toList();
          return fashions;
        } catch (e) {
          // Handle the case where response.data is not a List
          return (response.data['results'] as List)
              .map((json) => FashionModel.fromJson(json))
              .toList();
        }
      }
      throw response.data['details'];
    } on DioException catch (e) {
      // Проверяем на блокировку пользователя
      if (e.response?.statusCode == 400) {
        final responseData = e.response?.data;
        if (responseData is Map<String, dynamic>) {
          final status = responseData['status']?.toString();
          final detail = responseData['detail']?.toString() ?? '';
          if (status == '403' &&
              detail.toLowerCase().contains('заблокирован')) {
            throw UserBlockedException('Пользователь заблокирован');
          }
        }
      }

      if (e.response?.data['results'] != null) {
        return (e.response?.data['results'] as List)
            .map((json) => FashionModel.fromJson(json))
            .toList();
      }
      throw e.response?.data['details'];
    }
  }

  Future<FeedResult> getAllFashionsPagination(
      {required String nextPage}) async {
    try {
      Response response = await _dio.get(
        nextPage,
      );

      if (response.statusCode == 200 ||
          response.statusCode == 201 ||
          response.statusCode == 202) {
        FeedResult feedResult = FeedResult.fromJson(response.data);
        return feedResult;
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<FashionModel> createFashion({String? baseGender}) async {
    try {
      // POST /api/fashions/{profile_id}
      Response response = await _dio.post(
        await getFashionProfileEndpoint(),
        data: {
          'gender': baseGender,
        },
      );

      try {
        if (response.statusCode == 200 || response.statusCode == 201) {
          return FashionModel.fromJson(response.data);
        } else if (response.statusCode == 400) {
          FashionModel fashionModel = await getFashion(
            id: int.tryParse(response.data['detail'].last),
          );

          return fashionModel;
        }
      } catch (e) {
        throw e.toString();
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<bool> deleteDraft({required int id}) async {
    try {
      // DELETE /api/fashions/{profile_id}/{fashion_id}
      Response response = await _dio.delete(
        await getFashionByIdEndpoint(id),
      );

      return DeleteOperationHelper.handleDeleteResponse(response: response);
    } on DioException catch (e) {
      return DeleteOperationHelper.handleDeleteResponse(dioException: e);
    }
  }

  Future<bool> deleteFashionImage(int fashionId, int imageId) async {
    try {
      // DELETE /api/fashions/{profile_id}/{fashion_id}/photo/{photo_id}
      Response response = await _dio.delete(
        await getFashionDeletePhotoEndpoint(fashionId, imageId),
      );

      return DeleteOperationHelper.handleDeleteResponse(response: response);
    } on DioException catch (e) {
      return DeleteOperationHelper.handleDeleteResponse(dioException: e);
    }
  }

  Future<bool> deleteProduct(int fashionId, int productId) async {
    try {
      // DELETE /api/fashions/{profile_id}/products/{fashion_id}/{product_id}
      Response response = await _dio.delete(
        await getFashionProductByIdEndpoint(fashionId, productId),
      );

      return DeleteOperationHelper.handleDeleteResponse(response: response);
    } on DioException catch (e) {
      return DeleteOperationHelper.handleDeleteResponse(dioException: e);
    }
  }

  Future<FashionModel> updateFashion(
      {required FashionModel fashionModel}) async {
    try {
      // PATCH /api/fashions/{profile_id}/{fashion_id}
      Response response = await _dio.patch(
        await getFashionByIdEndpoint(fashionModel.id!),
        data: fashionModel.toJson(),
      );

      if (response.statusCode == 200 || response.statusCode == 201) {
        return FashionModel.fromJson(response.data);
      }
      throw response.data['detail'];
    } on DioException catch (e) {
      throw e.response?.data['detail'];
    }
  }

  Future<FashionModel> updateDescription(
      {required FashionModel fashionModel}) async {
    try {
      // PATCH /api/fashions/{profile_id}/{fashion_id}
      Response response = await _dio.patch(
        await getFashionByIdEndpoint(fashionModel.id!),
        data: fashionModel.toJson(),
      );

      if (response.statusCode == 200 || response.statusCode == 201) {
        return FashionModel.fromJson(response.data);
      }
      throw response.data['detail'];
    } on DioException catch (e) {
      throw e.response?.data['detail'];
    }
  }

  Future<FashionModel> publish({required FashionModel fashionModel}) async {
    try {
      // GET /api/fashions/{profile_id}/{fashion_id}/publish
      Response response = await _dio.get(
        await getFashionPublishEndpoint(fashionModel.id!),
      );

      if (response.statusCode == 200 || response.statusCode == 201) {
        return FashionModel.fromJson(response.data);
      }
      throw response.data['detail'];
    } on DioException catch (e) {
      throw e.response?.data['detail'];
    }
  }

  Future<TaskModel> addProduct(
      {required String url, required FashionModel fashionModel}) async {
    try {
      // POST /api/tasks/{profile_id}/parse-product/{fashion_id}
      Response response = await _dio.post(
        await getTaskParseProductEndpoint(fashionModel.id!),
        data: {
          "url": url,
        },
      );

      if (response.statusCode == 200 || response.statusCode == 201) {
        return TaskModel.fromJson(response.data);
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<TaskModel> checkProduct({required String taskId}) async {
    try {
      // GET /api/tasks/{profile_id}/{task_id}
      Response response = await _dio.get(
        await getTaskByIdEndpoint(taskId),
      );

      if (response.statusCode == 200 || response.statusCode == 201) {
        final responseData = response.data;

        // Проверяем статус задачи на наличие ошибок
        if (responseData['status'] == 'DUPLICATE' ||
            responseData['status'] == 'FAILED' ||
            responseData['status'] == 'ERROR') {
          // Пытаемся извлечь детальное сообщение об ошибке
          String errorMessage =
              'Попробуйте ещё раз или используйте другую ссылку';

          if (responseData['result'] != null) {
            final result = responseData['result'];

            // Приоритет: detail > error > стандартное сообщение
            if (result['detail'] != null &&
                result['detail'].toString().isNotEmpty) {
              errorMessage = result['detail'];
            } else if (result['error'] != null &&
                result['error'].toString().isNotEmpty) {
              errorMessage = result['error'];
            }
          }

          throw errorMessage;
        }

        return TaskModel.fromJson(responseData);
      }
      throw response.data['result']['detail'] ??
          'Попробуйте ещё раз или используйте другую ссылку';
    } on DioException catch (e) {
      // Обработка ошибок от сервера
      final errorData = e.response?.data;

      if (errorData != null) {
        if (errorData['result'] != null) {
          if (errorData['result']['detail'] != null) {
            throw errorData['result']['detail'];
          } else if (errorData['result']['error'] != null) {
            throw errorData['result']['error'];
          }
        }
        // Пытаемся извлечь максимум информации об ошибке
        if (errorData['details'] != null) {
          throw errorData['details'];
        } else if (errorData['detail'] != null) {
          throw errorData['detail'];
        } else if (errorData['error'] != null) {
          throw errorData['error'];
        }
      }

      throw 'Попробуйте ещё раз или используйте другую ссылку';
    } catch (e) {
      // Если это уже строка с ошибкой - пробрасываем дальше
      if (e is String) {
        rethrow;
      }
      throw 'Попробуйте ещё раз или используйте другую ссылку';
    }
  }

  Future<bool> cancelTask({required String taskId}) async {
    try {
      // DELETE /api/tasks/{profile_id}/{task_id}
      Response response = await _dio.delete(
        await getTaskByIdEndpoint(taskId),
      );

      return DeleteOperationHelper.handleDeleteResponse(response: response);
    } on DioException catch (e) {
      return DeleteOperationHelper.handleDeleteResponse(dioException: e);
    }
  }

  Future<FashionImage> uploadImage(
      {required int fashionId, required File file}) async {
    try {
      // Создаем FormData с изображением (уже сжатым в AddPostView)
      FormData formData = FormData.fromMap({
        'photo': await MultipartFile.fromFile(file.path),
      });

      // POST /api/fashions/{profile_id}/{fashion_id}/photo
      Response response = await _dio.post(
        await getFashionPhotoEndpoint(fashionId),
        options: Options(headers: {
          'Content-Type': 'multipart/form-data',
        }),
        data: formData,
      );

      if (response.statusCode == 200 || response.statusCode == 201) {
        return FashionImage.fromJson(response.data);
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'] ?? e.message;
    } catch (e) {
      throw e.toString();
    }
  }

  Future<String> likeFashion({required int fashionId}) async {
    try {
      // POST /api/fashions/{fashion_id}/like/{profile_id}
      final endpoint = await getFashionLikeEndpoint(fashionId);
      Response response = await _dio.post(endpoint);

      if (response.statusCode == 200 || response.statusCode == 201) {
        return response.data['detail'];
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<String> unlikeFashion({required int fashionId}) async {
    try {
      // DELETE /api/fashions/{fashion_id}/like/{profile_id}
      final endpoint = await getFashionLikeEndpoint(fashionId);
      Response response = await _dio.delete(endpoint);

      if (response.statusCode == 200 || response.statusCode == 201) {
        return response.data['detail'];
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<List<UserModel>> getFashionLikes({required int fashionId}) async {
    try {
      // GET /api/fashions/{fashion_id}/likes/{profile_id}
      final endpoint = await getFashionLikesEndpoint(fashionId);
      Response response = await _dio.get(endpoint);

      if (response.statusCode == 200 || response.statusCode == 201) {
        List<UserModel> users = (response.data as List).map((json) {
          UserModel userModel = UserModel.fromJson(json['user']);
          if (json['follow_status'] != null) {
            userModel.followStatus = json['follow_status'];
          }
          return userModel;
        }).toList();
        return users;
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<List<CommentModel>> getComments({required int fashionId}) async {
    try {
      // GET /api/fashions/{fashion_id}/{profile_id}/comments
      final endpoint = await getFashionCommentsEndpoint(fashionId);
      Response response = await _dio.get(endpoint);

      if (response.statusCode == 200 || response.statusCode == 201) {
        List<CommentModel> comments = (response.data as List)
            .map((json) => CommentModel.fromJson(json))
            .toList();
        return comments;
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<CommentModel> sendComment(
      {required int fashionId, required String text}) async {
    try {
      // POST /api/fashions/{fashion_id}/{profile_id}/comments
      final endpoint = await getFashionCommentsEndpoint(fashionId);
      Response response = await _dio.post(
        endpoint,
        data: {
          'comment': text,
        },
      );

      if (response.statusCode == 200 || response.statusCode == 201) {
        return CommentModel.fromJson(response.data);
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<bool> deleteComment(
      {required int fashionId, required int commentId, int? parentId}) async {
    try {
      String url;
      if (parentId != null) {
        // DELETE /api/fashions/{fashion_id}/{profile_id}/comments/answers/{parent_id}/{comment_id}
        url = await getFashionCommentAnswerByIdEndpoint(
            fashionId, parentId, commentId);
      } else {
        // DELETE /api/fashions/{fashion_id}/{profile_id}/comments/{comment_id}
        url = await getFashionCommentByIdEndpoint(fashionId, commentId);
      }

      Response response = await _dio.delete(url);

      if (response.statusCode == 200 ||
          response.statusCode == 201 ||
          response.statusCode == 204) {
        return true;
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<CommentModel> addAnswerToComment(
      {required int fashionId,
      required int commentId,
      required String text}) async {
    try {
      // POST /api/fashions/{fashion_id}/{profile_id}/comments/answers/{parent_id}
      final endpoint =
          await getFashionCommentAnswersEndpoint(fashionId, commentId);
      Response response = await _dio.post(
        endpoint,
        data: {
          'comment': text,
        },
      );

      if (response.statusCode == 200 || response.statusCode == 201) {
        return CommentModel.fromJson(response.data);
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<CommentModel> addAnswerToAnswer(
      {required int fashionId,
      required int commentId,
      required int parentId,
      required String text}) async {
    try {
      // PUT /api/fashions/{fashion_id}/{profile_id}/comments/answers/{parent_id}/{comment_id}
      final endpoint = await getFashionCommentAnswerByIdEndpoint(
          fashionId, parentId, commentId);
      Response response = await _dio.put(
        endpoint,
        data: {
          'comment': text,
        },
      );

      if (response.statusCode == 200 || response.statusCode == 201) {
        return CommentModel.fromJson(response.data);
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<List<CommentModel>> getAnswersToComment(
      {required int fashionId, required int parentId}) async {
    try {
      // GET /api/fashions/{fashion_id}/{profile_id}/comments/answers/{parent_id}
      final endpoint =
          await getFashionCommentAnswersEndpoint(fashionId, parentId);
      Response response = await _dio.get(endpoint);

      if (response.statusCode == 200 || response.statusCode == 201) {
        List<CommentModel> comments = (response.data as List)
            .map((json) => CommentModel.fromJson(json))
            .toList();
        return comments;
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<CommentModel> editComment({
    required int fashionId,
    required int commentId,
    required String text,
    int? parentId,
  }) async {
    try {
      String url;
      if (parentId != null) {
        // PUT /api/fashions/{fashion_id}/{profile_id}/comments/answers/{parent_id}/{comment_id}
        url = await getFashionCommentAnswerByIdEndpoint(
            fashionId, parentId, commentId);
      } else {
        // PUT /api/fashions/{fashion_id}/{profile_id}/comments/{comment_id}
        url = await getFashionCommentByIdEndpoint(fashionId, commentId);
      }

      Response response = await _dio.put(
        url,
        data: {
          'comment': text,
        },
      );

      if (response.statusCode == 200 || response.statusCode == 201) {
        return CommentModel.fromJson(response.data);
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }

  Future<bool> reportFashion({
    required int fashionId,
    required String reason,
    required String description,
  }) async {
    try {
      final String endpoint = await reportFashionEndpoint(fashionId);
      Response response = await _dio.post(endpoint, data: {
        "reason": reason,
        "description": description,
      });

      if (response.statusCode == 200 || response.statusCode == 201) {
        return true;
      }
      throw response.data['details'];
    } on DioException catch (e) {
      throw e.response?.data['details'];
    }
  }
}
```

## Сам View, из которого идет обращение в Cubit. Ничего не менял, не улучшал код. Показываю как есть
```dart
import 'dart:async';
import 'dart:io';

import 'package:cached_network_image/cached_network_image.dart';
import 'package:clooqs/core/api/endpoints.dart';
import 'package:clooqs/core/constants/assets.dart';
import 'package:clooqs/core/constants/colors.dart';
import 'package:clooqs/core/constants/styles.dart';
import 'package:clooqs/core/formatters/date.dart';
import 'package:clooqs/core/formatters/price.dart';
import 'package:clooqs/core/modal_sheets/sheets.dart';
import 'package:clooqs/core/models/fashion/fashion_model.dart';
import 'package:clooqs/core/router/navigation.dart';
import 'package:clooqs/features/home/presentation/views/fashion/bloc/fashion_cubit.dart';
import 'package:clooqs/features/home/presentation/views/fashion/widgets/product_info.dart';
import 'package:clooqs/features/home/presentation/views/profile/widgets/switch.dart';
import 'package:clooqs/features/widgets/custom_app_bar.dart';
import 'package:clooqs/features/widgets/custom_button.dart';
import 'package:clooqs/features/widgets/custom_text_field.dart';
import 'package:clooqs/features/widgets/loading_overlay.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_svg/svg.dart';
import 'package:image_picker/image_picker.dart';
import 'package:lottie/lottie.dart';
import 'package:url_launcher/url_launcher.dart';
import 'package:clooqs/core/constants/app_routes.dart';
import 'package:clooqs/core/utils/image_compression.dart';
import 'package:clooqs/features/auth/bloc/auth_cubit.dart';
import 'package:url_launcher/url_launcher_string.dart';

class AddPostView extends StatefulWidget {
  const AddPostView({super.key});

  @override
  State<AddPostView> createState() => _AddPostViewState();
}

class _AddPostViewState extends State<AddPostView> {
  FocusNode descriptionFocus = FocusNode();
  TextEditingController textEditingController = TextEditingController();
  Timer? _debounce;
  FashionModel? fashionModel;
  int timeInSearch = 0;

  Timer? timer;
  Timer? loadingTimer;

  final ScrollController _scrollController = ScrollController();

  bool waitingSnackBar = false;

  void _textControllerListener() {
    fashionModel?.description = textEditingController.text;
    _onTextChanged(textEditingController.text);
  }

  void _onTextChanged(String text) {
    if (_debounce?.isActive ?? false) _debounce!.cancel();

    _debounce = Timer(const Duration(seconds: 2), () {
      // 👇 Здесь отправляем на сервер
      if (kDebugMode) {}
      fashionCubit.updateDescription(fashionModel: fashionModel!);
    });
  }

  late FashionCubit fashionCubit;

  Product? currentProduct;
  String? currentTaskId; // ID текущей задачи парсинга товара

  Future<void> openBrowser(String url) async {
    Uri uri = Uri.parse(url);

    if (await canLaunchUrl(uri)) {
      await launchUrl(uri, mode: LaunchMode.externalApplication);
    } else {
      throw 'Не удалось открыть $url';
    }
  }

  Future<void> _pickImage() async {
    if (fashionModel == null) {
      final authCubit = context.read<AuthCubit>();
      fashionCubit.createFashion(authCubit.userModel.gender);
      return;
    }

    final pickedFile =
        await ImagePicker().pickImage(source: ImageSource.gallery);

    if (pickedFile != null) {
      final imageFile = File(pickedFile.path);

      // Автоматически сжимаем изображение под капотом
      final processedFile = await ImageCompressionUtil.compressImageAuto(
        file: imageFile,
        targetFileSize: 1024 * 1024, // 1MB
        maxWidth: 1080,
        maxHeight: 1920,
      );

      // Загружаем обработанное изображение
      fashionCubit.uploadImage(
        fashionId: fashionModel!.id!,
        image: processedFile,
      );
    }
  }

  _showProduct(List<Product?> product, int initialIndex) {
    AppSheets(context).showOldModal(
      header: 'Прикрепить товар',
      showButton: false,
      buttonTitle: currentProduct == null ? 'Прикрепить' : 'Готово',
      onSubmit: () {},
      child: Column(
        children: [
          ProductInfoWidget(
            product: product[initialIndex]!,
            onTap: () {
              currentProduct = null;
              fashionCubit.resetState();
            },
            preview: true,
          ),
          Container(height: 0.5, color: AppColors.grey),
          const SizedBox(height: 12),
          CustomButton(
            title: 'Подробнее',
            disabled: false,
            margin: 16,
            onTap: () {
              // authCubit.updateNickname(controller.text);
              // Navigator.pop(context);
              if (product[initialIndex]?.link != null) {
                // Открываем в внешнем браузере
                launchUrlString(
                  product[initialIndex]!.link!,
                  mode: LaunchMode.externalApplication,
                );
              }

              Navigator.pop(context);
            },
          ),
          const SizedBox(height: 12),
          if (fashionModel?.createdDate != null)
            Text(
              '${timeAgoShort(fashionModel!.createdDate!)} назад',
              style: AppTexts().sp14.copyWith(
                    color: AppColors.secondary,
                  ),
            ),
          const SizedBox(height: 34),
        ],
      ),
    );
  }

  _pickProduct() async {
    TextEditingController controller = TextEditingController();
    StreamController textController = StreamController();
    FocusNode focusNode = FocusNode();
    StreamController counterController = StreamController();

    controller.addListener(() {
      textController.sink.add(null);
    });
    fashionCubit.resetState();

    await AppSheets(context).showOldModal(
      header: 'Прикрепить товар',
      showButton: false,
      buttonTitle: currentProduct == null ? 'Прикрепить' : 'Готово',
      onSubmit: () {},
      controller: currentProduct == null ? controller : null,
      child: StreamBuilder(
        stream: textController.stream,
        builder: (context, snapshot) {
          return BlocBuilder<FashionCubit, FashionState>(
            buildWhen: (previous, current) {
              if (kDebugMode) {}
              if (current is AddProductSuccess) {
                currentProduct = Product(
                  id: current.productResult.id ?? 0,
                  photo: current.productResult.photo,
                  fashion: current.productResult.fashionId ?? 0,
                  name: current.productResult.name ?? '',
                  price: current.productResult.price,
                  link: current.productResult.link,
                );
              }
              // if (current is FashionSuccess) {
              //   currentProduct = current.fashion?.products
              //       ?.where((e) => e?.photo != null)
              //       .last;
              // }

              if (current is ProductLoading && previous is! FashionInitial) {
                // Ничего не делаем здесь - обработка в основном BlocBuilder
              }
              if (current is AddProductLongLoading) {
                loadingTimer?.cancel();
                timeInSearch = 0;
                loadingTimer = Timer.periodic(
                  const Duration(seconds: 1),
                  (timer) {
                    timeInSearch += 1;
                    counterController.sink.add(null);
                  },
                );
              }
              if (current is ProductLoadingError) {
                loadingTimer?.cancel();
                // AppSheets(context).showError(error: current.error);
              }

              return true;
            },
            builder: (context, state) {
              print('Текущее состояние: $state');
              return AnimatedContainer(
                duration: const Duration(milliseconds: 300),
                height: _getHeight(state),
                child: StreamBuilder(
                  stream: counterController.stream,
                  builder: (context, asyncSnapshot) {
                    return _buildProductBody(
                      state,
                      controller: controller,
                      focusNode: focusNode,
                      counterController: counterController,
                      textController: textController,
                      onTapProduct: (product) {
                        // _showProduct(product);
                      },
                    );
                  },
                ),
              );
            },
          );
        },
      ),
    );

    // Модальное окно закрыто - проверяем состояние задачи
    final currentState = fashionCubit.state;
    if (currentTaskId != null && currentState is AddProductLoading) {
      // Запрос ещё создаётся - отменяем
      await fashionCubit.cancelProductTask(taskId: currentTaskId!);
      if (kDebugMode) {
        print('🛑 Задача отменена (запрос создавался): $currentTaskId');
      }
      currentTaskId = null;
    } else if (currentTaskId != null &&
        (currentState is ProductLoading ||
            currentState is AddProductLongLoading)) {
      // Запрос уже отправлен - НЕ отменяем, слушаем результат на главном экране
      if (kDebugMode) {
        print('🔄 Модалка закрыта, задача продолжается в фоне: $currentTaskId');
      }
    } else if (kDebugMode) {
      print('✅ Модалка закрыта, задача завершена или не запускалась');
    }

    fashionCubit.resetState();
    timer?.cancel();
    descriptionFocus.unfocus();

    loadingTimer?.cancel();
    currentProduct = null;
  }

  double _getHeight(FashionState state) {
    if (state is ProductLoading) {
      return 277 - 34;
    } else if (currentProduct != null) {
      double screenWidth = MediaQuery.of(context).size.width;
      double cardHeight = screenWidth - 32;
      double extraHeight = cardHeight + 24 + 140 + 53 - 32;
      if ((currentProduct?.name ?? '').length > 50) {
        extraHeight += 20;
      }
      return extraHeight;
    } else if (state is ProductLoadingError) {
      double screenWidth = MediaQuery.of(context).size.width;
      double cardHeight = screenWidth - 32;
      double extraHeight = cardHeight + 24 + 140 + 46 + 16 - 32;
      return extraHeight;
    } else if (state is AddProductLongLoading ||
        state is AddProductTooLongLoading) {
      double screenWidth = MediaQuery.of(context).size.width;
      double cardHeight = screenWidth - 32;
      double extraHeight = cardHeight + 24 + 140 + 46 + 16 - 32;
      return extraHeight;
    }
    return 277 - 34;
  }

  Widget _buildProductBody(
    FashionState state, {
    required TextEditingController controller,
    required FocusNode focusNode,
    required StreamController textController,
    required StreamController counterController,
    required Function(Product) onTapProduct,
  }) {
    if (state is AddProductLongLoading || state is AddProductTooLongLoading) {
      double screenWidth = MediaQuery.of(context).size.width;
      double cardHeight = screenWidth - 32;
      return Container(
        height: 510,
        padding: const EdgeInsets.symmetric(horizontal: 16),
        child: Column(
          children: [
            Stack(
              children: [
                Align(
                  alignment: Alignment.topCenter,
                  child: SvgPicture.asset(
                    AppIcons.progress,
                    height: cardHeight,
                    width: double.infinity,
                  ),
                ),
                Container(
                  height: cardHeight,
                  width: cardHeight,
                  margin: const EdgeInsets.symmetric(horizontal: 16),
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      SizedBox(
                        height: 128,
                        child: Stack(
                          children: [
                            Center(
                              child: LottieBuilder.asset(
                                AppAnimations.productLoading,
                                height: 128,
                              ),
                            ),
                            // Align(
                            //   alignment: Alignment.bottomCenter,
                            //   child: Padding(
                            //     padding: const EdgeInsets.only(bottom: 20),
                            //     child: SvgPicture.asset(
                            //       AppIcons.bag,
                            //       height: 128,
                            //     ),
                            //   ),
                            // ),
                          ],
                        ),
                      ),
                      const SizedBox(height: 15.5),
                      Text(
                        'Поиск и загрузка товара...',
                        style: AppTexts().h4,
                        textAlign: TextAlign.center,
                      ),
                      const SizedBox(height: 8),
                      Text(
                        'Загружаем фото, описание и стоимость товара, пожалуйста подождите',
                        style: AppTexts().sp16,
                        textAlign: TextAlign.center,
                      ),
                      const SizedBox(height: 8),
                      SizedBox(
                        height: 24,
                        child: Text(
                          timeInSearch.toString(),
                          style: AppTexts().h2,
                        ),
                      ),
                    ],
                  ),
                )
              ],
            ),
            if (state is AddProductTooLongLoading) ...[
              const SizedBox(height: 24),
              Container(
                padding: const EdgeInsets.symmetric(
                  horizontal: 12,
                  vertical: 16,
                ),
                width: double.infinity,
                decoration: BoxDecoration(
                  color: AppColors.dark,
                  borderRadius: BorderRadius.circular(12),
                ),
                child: Column(
                  children: [
                    SvgPicture.asset(
                      AppIcons.loading,
                      height: 28,
                      width: 28,
                    ),
                    const SizedBox(height: 12),
                    Text(
                      'Наш ИИ старается как не в себя',
                      style: AppTexts().h4.copyWith(
                            color: AppColors.white,
                          ),
                    ),
                    const SizedBox(height: 8),
                    Text(
                      'Иногда процесс занимает до 15 сек,\nв редких случаях до 35 сек',
                      style: AppTexts().sp16.copyWith(
                            color: AppColors.white,
                          ),
                      textAlign: TextAlign.center,
                    ),
                  ],
                ),
              ),
              // const SizedBox(height: 46),
            ],
          ],
        ),
      );
    }
    if (state is ProductLoadingError) {
      if (kDebugMode) {}
      double screenWidth = MediaQuery.of(context).size.width;
      double cardHeight = screenWidth - 32;
      return Container(
        height: 510,
        child: Column(
          children: [
            Stack(
              children: [
                Align(
                  alignment: Alignment.topCenter,
                  child: SvgPicture.asset(
                    AppIcons.progress,
                    height: cardHeight,
                    width: double.infinity,
                  ),
                ),
                Container(
                  height: cardHeight,
                  width: cardHeight,
                  margin: const EdgeInsets.symmetric(horizontal: 16),
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      SizedBox(
                        height: 28,
                        child: Stack(
                          children: [
                            Center(
                              child: SvgPicture.asset(AppIcons.attentionRed),
                            ),
                            // Align(
                            //   alignment: Alignment.bottomCenter,
                            //   child: Padding(
                            //     padding: const EdgeInsets.only(bottom: 20),
                            //     child: SvgPicture.asset(
                            //       AppIcons.bag,
                            //       height: 128,
                            //     ),
                            //   ),
                            // ),
                          ],
                        ),
                      ),
                      const SizedBox(height: 15.5),
                      Text(
                        'Не удалось прикрепить товар',
                        style: AppTexts().h4,
                        textAlign: TextAlign.center,
                      ),
                      const SizedBox(height: 8),
                      Padding(
                        padding: const EdgeInsets.symmetric(horizontal: 16),
                        child: Text(
                          state.error,
                          style: AppTexts().sp16,
                          textAlign: TextAlign.center,
                        ),
                      ),
                    ],
                  ),
                )
              ],
            ),
            const Expanded(child: const SizedBox(height: 120)),
            Container(height: 0.5, color: AppColors.grey),
            const SizedBox(height: 16),
            CustomButton(
              title: 'Попробовать ещё раз',
              disabled: false,
              onTap: () {
                fashionCubit.resetState();
                Future.delayed(const Duration(milliseconds: 300), () {
                  setState(() {
                    focusNode.requestFocus();
                    textController.sink.add(null);
                  });
                });
              },
            ),
          ],
        ),
      );
    }
    if (currentProduct != null) {
      currentProduct?.link ??= controller.text;

      return Container(
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            ProductInfoWidget(
              product: currentProduct!,
              onTap: () {
                if (currentProduct?.link != null) {
                  // Открываем в внешнем браузере
                  launchUrlString(
                    currentProduct!.link!,
                    mode: LaunchMode.externalApplication,
                  );
                }
              },
              preview: false,
            ),
            // Container(
            //   height: 343,
            //   decoration: BoxDecoration(
            //     border: Border.all(color: AppColors.grey, width: 0.5),
            //     borderRadius: BorderRadius.circular(12),
            //     image: DecorationImage(
            //       image: CachedNetworkImageProvider(
            //         Endpoints.mediaUrl + (currentProduct?.photo ?? ''),
            //       ),
            //       fit: BoxFit.cover,
            //     ),
            //   ),
            //   // child: Image.network(
            //   //   Endpoints.mediaUrl + (product.photo ?? ''),
            //   //   width: double.infinity,
            //   //   height: 343,
            //   //   fit: BoxFit.cover,
            //   // ),
            // ),
            // const SizedBox(height: 24),
            // Text(
            //   currentProduct?.name ?? '',
            //   style: AppTexts().sp14.copyWith(overflow: TextOverflow.ellipsis),
            //   maxLines: 1,
            // ),
            // const SizedBox(height: 4),
            // formattedPrice(currentProduct?.price ?? ''),
            const Expanded(child: SizedBox()),
            Container(height: 0.5, color: AppColors.grey),
            const SizedBox(height: 16),
            CustomButton(
              title: 'Готово',
              disabled: false,
              onTap: () {
                // authCubit.updateNickname(controller.text);
                // Navigator.pop(context);

                currentProduct = null;
                Navigator.pop(context);
                fashionCubit.resetState();
              },
            ),
          ],
        ),
      );
    }
    return Container(
      // padding: const EdgeInsets.symmetric(horizontal: 16),
      // height: 290,
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                CustomTextField(
                  title: 'Вставьте ссылку на товар',
                  hint: focusNode.hasFocus ? '' : 'https://',
                  primary: false,
                  focusNode: focusNode,
                  controller: controller,
                  waitingSnackBar: waitingSnackBar,
                  enabled:
                      state is! ProductLoading && state is! AddProductLoading,
                  onInsert: () {
                    HapticFeedback.mediumImpact();
                    Clipboard.getData('text/plain').then((value) {
                      if (value != null &&
                          value.text != null &&
                          value.text!.trim().isNotEmpty) {
                        controller.text = value.text!.replaceAll('\n', '');
                        focusNode.requestFocus();
                        controller.selection = TextSelection.fromPosition(
                          TextPosition(offset: controller.text.length),
                        );
                      } else {
                        setState(() {
                          waitingSnackBar = true;
                          textController.sink.add(null);
                        });
                        AppSheets(context).showCustomSnackBar(
                          message:
                              'Сначала необходимо скопировать ссылку на товар',
                          onAnimationEnd: () {
                            setState(() {
                              waitingSnackBar = false;
                              textController.sink.add(null);
                            });
                          },
                        );
                      }
                    });
                  },
                  maxLines: 2,
                ),
                const SizedBox(height: 4),
                Text(
                  'Скопируйте ссылку с помощью функции\n«Поделиться → Скопировать» на сайте,\nв приложении розничного магазина\nили маркетплейса (Я.Маркет, Озон, Вайлдберриз)',
                  style: AppTexts().sp14.copyWith(
                        color: AppColors.secondary,
                      ),
                  textAlign: TextAlign.start,
                ),
                const SizedBox(height: 36),
              ],
            ),
          ),
          // const Expanded(child: SizedBox()),
          Container(height: 0.5, color: AppColors.grey),
          const SizedBox(height: 12),
          CustomButton(
            title: currentProduct == null ? 'Прикрепить' : 'Готово',
            disabled: controller.value.text.isEmpty == true,
            isLoading: state is ProductLoading || state is AddProductLoading,
            margin: 16,
            onTap: () {
              if (state is ProductLoadingError) {
                fashionCubit.resetState();
                return;
              }
              // authCubit.updateNickname(controller.text);
              // Navigator.pop(context);
              if (fashionModel != null) {
                if (currentProduct == null) {
                  focusNode.unfocus();

                  fashionCubit.addProduct(
                    url: controller.text,
                    fashionModel: fashionModel!,
                  );
                } else {
                  currentProduct = null;
                  Navigator.pop(context);
                }
              }
            },
          ),
          // const SizedBox(height: 34),
        ],
      ),
    );
  }

  /// Проверяет и восстанавливает null элемент в конце списка товаров для кнопки добавления
  void _ensureNullElementAtEnd() {
    if (fashionModel?.products != null) {
      final realProductCount =
          fashionModel!.products!.where((p) => p != null).length;
      final hasNullAtEnd = fashionModel!.products!.isNotEmpty &&
          fashionModel!.products!.last == null;

      // Всегда должен быть null элемент в конце
      // Если товаров < 5, это будет кнопка добавления
      // Если товаров = 5, это будет блок "максимум достигнут"
      if (!hasNullAtEnd) {
        fashionModel!.products!.add(null);
        if (kDebugMode) {
          print(
              '🛍️ Restored null element at end. Real products count: $realProductCount');
        }
      }
    }
  }

  @override
  void initState() {
    fashionCubit = context.read<FashionCubit>();

    textEditingController.addListener(_textControllerListener);
    descriptionFocus.addListener(() {
      if (descriptionFocus.hasFocus) {
        print('Фокус на описании - скроллим');
        Future.delayed(const Duration(milliseconds: 600), () {
          Scrollable.ensureVisible(
            descriptionFocus.context!,
            duration: const Duration(milliseconds: 300),
            curve: Curves.easeInOut,
            alignment: 0.85,
          );
        });
      }
    });

    super.initState();
  }

  @override
  void dispose() {
    _debounce?.cancel();
    textEditingController.removeListener(_textControllerListener);
    textEditingController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (kDebugMode) {}
    return Scaffold(
      backgroundColor: AppColors.white,
      appBar: CustomAppBar(
        close: true,
        back: false,
        title: 'Новый образ',
        onClose: () {
          AppSheets(context).closeFashionAlert(
            onTapDraft: () {
              Navigator.pop(context);
            },
            onTapDelete: () {
              if (fashionModel?.id != null) {
                fashionCubit.deleteDraft(
                    id: fashionModel!.id!, context: context);
              }
            },
          );
        },
      ),
      body: GestureDetector(
        onTap: () {
          FocusScope.of(context).unfocus();
        },
        child: BlocBuilder<FashionCubit, FashionState>(
            buildWhen: (previous, current) {
          if (current is ProductLoading && previous is FashionInitial) {
            fashionCubit.resetState();
            return false;
          }
          // Обработка загрузки товара - запускаем таймер проверки статуса
          if (current is ProductLoading && previous is! FashionInitial) {
            currentTaskId = current.taskModel.taskId;
            if (kDebugMode) {
              print('🔄 Запуск проверки статуса задачи: $currentTaskId');
            }

            if (fashionModel?.id != null) {
              Future.delayed(
                const Duration(seconds: 3),
                () {
                  if (currentTaskId == current.taskModel.taskId) {
                    fashionCubit.checkProductStatus(
                      taskId: current.taskModel.taskId,
                      fashionId: fashionModel!.id!,
                    );
                    timer = Timer.periodic(
                      const Duration(seconds: 7),
                      (timer) {
                        if (currentTaskId == current.taskModel.taskId) {
                          fashionCubit.checkProductStatus(
                            taskId: current.taskModel.taskId,
                            fashionId: fashionModel!.id!,
                          );
                        } else {
                          timer.cancel();
                        }
                      },
                    );
                  }
                },
              );
            }
            return false; // Не перестраиваем UI при ProductLoading в основном экране
          }

          if (current is FashionSuccess) {
            if (kDebugMode) {}
            fashionModel = current.fashion;
            fashionModel?.fashionImages?.add(null);
            if (fashionModel?.products != null) {
              // Добавляем null элемент только если еще нет или если последний элемент не null
              if (fashionModel!.products!.isEmpty ||
                  fashionModel!.products!.last != null) {
                fashionModel!.products!.add(null);
              }
            }
            textEditingController.text = fashionModel?.description ?? '';
            fashionModel?.gender ??= context.read<AuthCubit>().userModel.gender;

            // Дополнительная проверка для восстановления null элемента через небольшую задержку
            WidgetsBinding.instance.addPostFrameCallback((_) {
              if (mounted) {
                _ensureNullElementAtEnd();
              }
            });
            return true;
          }
          // if (current is ProductLoadingError) {
          //   timer?.cancel();
          //   loadingTimer?.cancel();
          //   return true;
          // }
          // if (current is AddProductSuccess) {
          //   fashionModel?.products?.add(
          //     Product(
          //       id: current.productResult.id ?? 0,
          //       photo: current.productResult.photo,
          //       fashion: current.productResult.fashionId ?? 0,
          //       name: current.productResult.name ?? '',
          //       price: current.productResult.price,
          //     ),
          //   );
          //   return true;
          // }
          if (current is DeleteDraftSuccess) {
            Navigator.pop(context);
          }
          if (current is AddProductSuccess) {
            // Задача завершена успешно
            currentTaskId = null;
            timer?.cancel();
            loadingTimer?.cancel();

            if (fashionModel?.products != null) {
              if (kDebugMode) {
                print(
                    '🛍️ Before adding product: ${fashionModel!.products!.map((p) => p?.name ?? 'null').toList()}');
              }

              // Удаляем null элемент (кнопку добавления) если он есть в конце
              if (fashionModel!.products!.isNotEmpty &&
                  fashionModel!.products!.last == null) {
                fashionModel!.products!.removeLast();
                if (kDebugMode) {
                  print('🛍️ Removed null element from end');
                }
              }

              // Добавляем новый товар в конец списка
              final newProduct = Product(
                id: current.productResult.id ?? 0,
                photo: current.productResult.photo,
                fashion: current.productResult.fashionId ?? 0,
                name: current.productResult.name ?? '',
                price: current.productResult.price,
                link: current.productResult.link,
              );
              fashionModel!.products!.add(newProduct);

              if (kDebugMode) {
                print('🛍️ Added product: ${newProduct.name}');
              }

              // Всегда добавляем null элемент обратно в конец
              // Если товаров < 5, это будет кнопка добавления
              // Если товаров = 5, это будет блок "максимум достигнут"
              final realProductCount =
                  fashionModel!.products!.where((p) => p != null).length;
              fashionModel!.products!.add(null);
              if (kDebugMode) {
                print(
                    '🛍️ Added null element back to end. Real products count: $realProductCount');
              }

              if (kDebugMode) {
                print(
                    '🛍️ After adding product: ${fashionModel!.products!.map((p) => p?.name ?? 'null').toList()}');
              }
            }
            Future.delayed(const Duration(milliseconds: 100), () {
              _scrollController.animateTo(
                _scrollController.position.maxScrollExtent,
                duration: const Duration(milliseconds: 300),
                curve: Curves.easeOut,
              );
            });
            return true;
          }
          if (current is FashionPublicateSuccess) {
            return false;
            // Не закрываем AddPostView здесь — закрытие только после прекеша в FeedView!
          }
          if (current is FashionPublicateError) {
            AppSheets(context).showError(error: current.error);
          }
          if (current is FashionImageSuccess) {
            if (fashionModel != null) {
              fashionModel!.fashionImages!.insert(
                fashionModel!.fashionImages!.length - 1,
                current.fashionImage,
              );
            }
            // Не закрываем AddPostView — закрытие только после публикации!
            return true;
          }
          if (current is FashionPhotoDeleted) {
            fashionModel?.fashionImages?.removeWhere(
              (e) => e?.id == current.photoId,
            );
            return true;
          }

          if (current is ProductLoadingError) {
            // Задача завершена с ошибкой
            currentTaskId = null;
            timer?.cancel();
            loadingTimer?.cancel();
          }

          if (current is ProductDeleted) {
            if (fashionModel?.products != null) {
              if (kDebugMode) {
                print(
                    '🛍️ Before deleting product: ${fashionModel!.products!.map((p) => p?.name ?? 'null').toList()}');
              }

              fashionModel!.products!.removeWhere(
                (e) => e?.id == current.productId,
              );

              // Убеждаемся, что есть null элемент в конце, если товаров меньше 5
              final realProductCount =
                  fashionModel!.products!.where((p) => p != null).length;
              final hasNullAtEnd = fashionModel!.products!.isNotEmpty &&
                  fashionModel!.products!.last == null;

              if (realProductCount < 5 && !hasNullAtEnd) {
                fashionModel!.products!.add(null);
                if (kDebugMode) {
                  print(
                      '🛍️ Added null element after deletion. Real products count: $realProductCount');
                }
              }

              if (kDebugMode) {
                print(
                    '🛍️ After deleting product: ${fashionModel!.products!.map((p) => p?.name ?? 'null').toList()}');
              }
            }
            return true;
          }
          if (current is CreateFashionLoading) {
            return true;
          }
          if (current is AllFashionSuccess) {
            return false;
          }
          return true;
        }, builder: (context, state) {
          print('🛍️ Building AddPostView with state: $state');
          // Проверяем и восстанавливаем null элемент при каждой перестройке
          WidgetsBinding.instance.addPostFrameCallback((_) {
            if (mounted) {
              _ensureNullElementAtEnd();
            }
          });

          return Stack(
            children: [
              CustomScrollView(
                controller: _scrollController,
                slivers: [
                  const SliverToBoxAdapter(child: SizedBox(height: 16)),
                  SliverToBoxAdapter(
                    child: Container(
                      // height: 64,
                      padding: const EdgeInsets.symmetric(
                          horizontal: 16, vertical: 12),
                      child: Row(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Expanded(
                            child: Column(
                              crossAxisAlignment: CrossAxisAlignment.start,
                              children: [
                                Row(
                                  children: [
                                    Text(
                                      'Фото образа',
                                      style: AppTexts().h4,
                                    ),
                                    Container(
                                      width: 16,
                                      height: 16,
                                      alignment: Alignment.center,
                                      child: Container(
                                        height: 4,
                                        width: 4,
                                        decoration: const BoxDecoration(
                                          color: AppColors.error,
                                          shape: BoxShape.circle,
                                        ),
                                      ),
                                    ),
                                  ],
                                ),
                                const SizedBox(height: 4),
                                Text(
                                  'Добавьте фото образа человека. Не подходят: фото вещей без модели.',
                                  style: AppTexts().sp16,
                                ),
                              ],
                            ),
                          ),
                          const SizedBox(width: 12),
                          GestureDetector(
                            onTap: () {
                              HapticFeedback.mediumImpact();
                              AppSheets(context)
                                  .showOldModal(
                                header: 'Что такое фото образа',
                                child: Column(
                                  children: [
                                    Container(
                                      padding: const EdgeInsets.symmetric(
                                        horizontal: 16,
                                      ),
                                      child: Text(
                                        'Фото образа — это снимок, где виден человек, он показывает, как вещи сидят и сочетаются друг с другом в реальности',
                                        style: AppTexts().sp16,
                                      ),
                                    ),
                                    const SizedBox(height: 16),
                                    SvgPicture.asset(
                                      AppIcons.fashionInfo,
                                      width: MediaQuery.of(context).size.width,
                                      fit: BoxFit.fitWidth,
                                      // height: 214,
                                    ),
                                    Container(
                                      padding: const EdgeInsets.symmetric(
                                          horizontal: 16, vertical: 12),
                                      child: RichText(
                                        text: TextSpan(
                                          text:
                                              'Не подходят: фото вещей на вешалке, только предметы одежды, обуви и аксессуаров без человека.\n\n',
                                          style: AppTexts().sp16,
                                          children: [
                                            TextSpan(
                                              text:
                                                  'Товары (одежду, обувь и аксессуары) можно прикрепить отдельно в разделе ниже',
                                              style: AppTexts().h4,
                                            )
                                          ],
                                        ),
                                      ),
                                    ),
                                    const SizedBox(height: 24),
                                  ],
                                ),
                                buttonTitle: 'Понятно',
                                onSubmit: () {
                                  Navigator.pop(context);
                                },
                              )
                                  .then((value) {
                                descriptionFocus.unfocus();
                              });
                            },
                            child: SvgPicture.asset(AppIcons.info),
                          )
                        ],
                      ),
                    ),
                  ),
                  const SliverToBoxAdapter(child: SizedBox(height: 4)),
                  // Padding(
                  //   padding: EdgeInsets.symmetric(horizontal: 16),
                  //   child: Row(
                  //     mainAxisAlignment: MainAxisAlignment.center,
                  //     children: [
                  //       image(null),
                  //       SizedBox(width: 8),
                  //       image(null),
                  //     ],
                  //   ),
                  // )
                  SliverPadding(
                    padding:
                        const EdgeInsets.symmetric(horizontal: 16, vertical: 0),
                    sliver: SliverGrid(
                      gridDelegate:
                          const SliverGridDelegateWithFixedCrossAxisCount(
                        crossAxisCount: 2,
                        crossAxisSpacing: 8,
                        mainAxisSpacing: 8,
                        childAspectRatio: 165 / 296,
                      ),
                      delegate: SliverChildBuilderDelegate(
                        (context, index) {
                          final images = fashionModel?.fashionImages;
                          final fashionImage =
                              (images != null && index < images.length)
                                  ? images[index]
                                  : null;
                          return image(
                            fashionImage,
                            state is FashionImageLoading,
                            fashionState: state,
                          );
                        },
                        childCount: fashionModel?.fashionImages?.length ?? 1,
                      ),
                    ),
                  ),
                  const SliverToBoxAdapter(child: SizedBox(height: 16)),
                  SliverToBoxAdapter(
                    child: Container(
                      margin: const EdgeInsets.symmetric(horizontal: 16),
                      // height: 78,
                      width: double.infinity,
                      decoration: BoxDecoration(
                        borderRadius: BorderRadius.circular(12),
                        color: AppColors.grey,
                      ),
                      padding: const EdgeInsets.symmetric(
                        horizontal: 12,
                        vertical: 12,
                      ),
                      child: Row(
                        children: [
                          SvgPicture.asset(
                            AppIcons.attention,
                            height: 28,
                          ),
                          const SizedBox(width: 12),
                          Expanded(
                            child: Text(
                              'На фото не должно быть алкоголя, табака, оружия. Не добавляйте картинки с водяными знаками и рекламу.',
                              style: AppTexts().sp14.copyWith(
                                    fontWeight: FontWeight.w400,
                                  ),
                            ),
                          )
                        ],
                      ),
                    ),
                  ),
                  const SliverToBoxAdapter(child: SizedBox(height: 24)),
                  SliverToBoxAdapter(
                    child: Padding(
                      padding: const EdgeInsets.only(left: 16),
                      child: Row(
                        children: [
                          Text('Пол образа', style: AppTexts().h4),
                          Container(
                            width: 16,
                            height: 16,
                            alignment: Alignment.center,
                            child: Container(
                              height: 4,
                              width: 4,
                              decoration: const BoxDecoration(
                                color: AppColors.error,
                                shape: BoxShape.circle,
                              ),
                            ),
                          ),
                        ],
                      ),
                    ),
                  ),
                  const SliverToBoxAdapter(child: SizedBox(height: 8)),
                  SliverToBoxAdapter(
                    child: Padding(
                      padding: const EdgeInsets.symmetric(horizontal: 16),
                      child: Row(
                        children: [
                          GestureDetector(
                            onTap: () {
                              HapticFeedback.mediumImpact();
                              fashionModel?.gender = 'female';
                              setState(() {});
                              // setState(() {
                              //   gender = gender == Gender.female
                              //       ? null
                              //       : Gender.female;
                              // });
                            },
                            child: Container(
                              height: 36,
                              padding: const EdgeInsets.symmetric(
                                horizontal: 12,
                              ),
                              decoration: BoxDecoration(
                                color: fashionModel?.gender == 'female'
                                    ? AppColors.primary
                                    : AppColors.grey,
                                borderRadius: BorderRadius.circular(18),
                              ),
                              alignment: Alignment.center,
                              child: Container(
                                // height: 20,
                                alignment: Alignment.center,
                                child: Text(
                                  'Женский',
                                  style: AppTexts().sp16.copyWith(
                                        color: fashionModel?.gender == 'female'
                                            ? AppColors.white
                                            : AppColors.dark,
                                      ),
                                  textHeightBehavior: const TextHeightBehavior(
                                    applyHeightToFirstAscent:
                                        true, // Не влияет на верхнюю часть текста
                                    applyHeightToLastDescent:
                                        false, // Не влияет на нижнюю часть текста
                                  ),
                                ),
                              ),
                            ),
                          ),
                          const SizedBox(width: 8),
                          GestureDetector(
                            onTap: () {
                              HapticFeedback.mediumImpact();
                              fashionModel?.gender = 'male';
                              setState(() {});
                              // setState(() {
                              //   gender =
                              //       gender == Gender.male ? null : Gender.male;
                              // });
                            },
                            child: Container(
                              height: 36,
                              padding:
                                  const EdgeInsets.symmetric(horizontal: 12),
                              decoration: BoxDecoration(
                                color: fashionModel?.gender == 'male'
                                    ? AppColors.primary
                                    : AppColors.grey,
                                borderRadius: BorderRadius.circular(18),
                              ),
                              alignment: Alignment.center,
                              child: Text(
                                'Мужской',
                                style: AppTexts().sp16.copyWith(
                                      color: fashionModel?.gender == 'male'
                                          ? AppColors.white
                                          : AppColors.dark,
                                    ),
                                textHeightBehavior: const TextHeightBehavior(
                                  applyHeightToFirstAscent:
                                      true, // Не влияет на верхнюю часть текста
                                  applyHeightToLastDescent:
                                      false, // Не влияет на нижнюю часть текста
                                ),
                              ),
                            ),
                          ),
                        ],
                      ),
                    ),
                  ),
                  const SliverToBoxAdapter(child: SizedBox(height: 24)),
                  SliverToBoxAdapter(
                    child: Padding(
                      padding: const EdgeInsets.symmetric(horizontal: 16),
                      child: CustomTextField(
                        title: 'Описание образа',
                        expand: true,
                        bold: true,
                        hint: 'Добавить...',
                        showCounter: true,
                        error: textEditingController.text.length > 300
                            ? 'Максимум 300 символов'
                            : null,
                        controller: textEditingController,
                        focusNode: descriptionFocus,
                        textCapitalization: TextCapitalization.sentences,
                        textInputType: TextInputType.text,
                        onChanged: (p0) {
                          setState(() {});
                        },
                        maxLength: 300,
                      ),
                    ),
                  ),
                  // const SliverToBoxAdapter(child: SizedBox(height: 4)),
                  // SliverToBoxAdapter(
                  //   child: Container(
                  //     margin: const EdgeInsets.symmetric(horizontal: 16),
                  //     height: 13,
                  //     width: double.infinity,
                  //     child: Text('${textEditingController.text.length}/300',
                  //         textAlign: TextAlign.end,
                  //         style: AppTexts().sp14.copyWith(
                  //               color: textEditingController.text.length == 300
                  //                   ? AppColors.dark
                  //                   : textEditingController.text.length > 300
                  //                       ? AppColors.error
                  //                       : AppColors.greyDark,
                  //               fontSize: 12,
                  //               fontWeight: FontWeight.w400,
                  //             )),
                  //   ),
                  // ),
                  const SliverToBoxAdapter(child: SizedBox(height: 12)),
                  SliverToBoxAdapter(
                    child: Padding(
                      padding: const EdgeInsets.symmetric(horizontal: 16),
                      child: SizedBox(
                        height: 76,
                        child: SettingSwitch(
                          title: 'Комментарии к образу',
                          description: (fashionModel?.allowComments ?? true)
                              ? 'Включены'
                              : 'Выключены',
                          bold: true,
                          value: (fashionModel?.allowComments ?? true),
                          onChanged: (value) {
                            fashionModel?.allowComments = value;
                            fashionCubit.updateDescription(
                              fashionModel: fashionModel!,
                            );
                            setState(() {});
                          },
                        ),
                      ),
                    ),
                  ),
                  const SliverToBoxAdapter(child: SizedBox(height: 12)),
                  SliverToBoxAdapter(
                    child: Container(
                      padding: const EdgeInsets.symmetric(horizontal: 16),
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Row(
                            mainAxisAlignment: MainAxisAlignment.spaceBetween,
                            children: [
                              Text(
                                'Товары из образа',
                                style: AppTexts().h4,
                              ),
                              GestureDetector(
                                onTap: () {
                                  HapticFeedback.mediumImpact();
                                  AppSheets(context)
                                      .showOldModal(
                                    header: 'Что такое товары из образа',
                                    child: Column(
                                      children: [
                                        Container(
                                          padding: const EdgeInsets.symmetric(
                                            horizontal: 16,
                                          ),
                                          child: Text(
                                            'Укажите ссылку на карточку товара (одежда, обувь, аксессуары) и в образе будет прикреплен товар с ссылкой на продавца',
                                            style: AppTexts().sp16,
                                          ),
                                        ),
                                        const SizedBox(height: 16),
                                        Container(
                                          padding: const EdgeInsets.symmetric(
                                            horizontal: 16,
                                            vertical: 12,
                                          ),
                                          child: Column(
                                            crossAxisAlignment:
                                                CrossAxisAlignment.start,
                                            children: [
                                              Text(
                                                'Как прикрепить товар к образу',
                                                style: AppTexts().h4,
                                              ),
                                              const SizedBox(height: 8),
                                              Row(
                                                crossAxisAlignment:
                                                    CrossAxisAlignment.start,
                                                children: [
                                                  Text(
                                                    '1. ',
                                                    style: AppTexts().sp16,
                                                  ),
                                                  Expanded(
                                                    child: Text(
                                                      'Скопируйте ссылку на карточку товара с помощью функции «Поделиться → Скопировать» на сайте, в приложении розничного магазина или маркетплейса (Я.Маркет, Озон, Вайлдберриз)',
                                                      style: AppTexts().sp16,
                                                    ),
                                                  ),
                                                ],
                                              ),
                                              Row(
                                                crossAxisAlignment:
                                                    CrossAxisAlignment.start,
                                                children: [
                                                  Text(
                                                    '2. ',
                                                    style: AppTexts().sp16,
                                                  ),
                                                  Expanded(
                                                    child: Text(
                                                      'Вставьте в поле добавления и нажмите «Прикрепить» — товар (фото, наименование и цена) будет добавлен автоматически',
                                                      style: AppTexts().sp16,
                                                    ),
                                                  ),
                                                ],
                                              ),
                                            ],
                                          ),
                                        ),
                                        if (context
                                                .read<AuthCubit>()
                                                .isBusinessAccountMode ==
                                            false) ...[
                                          Container(
                                            padding: const EdgeInsets.symmetric(
                                              vertical: 12,
                                              horizontal: 16,
                                            ),
                                            width: double.infinity,
                                            child: Column(
                                              crossAxisAlignment:
                                                  CrossAxisAlignment.start,
                                              children: [
                                                Text(
                                                  'Для магазина, бренда или продавца',
                                                  style: AppTexts().h4,
                                                ),
                                                const SizedBox(height: 8),
                                                GestureDetector(
                                                  onTap: () {
                                                    HapticFeedback
                                                        .mediumImpact();
                                                    AuthCubit authCubit =
                                                        context
                                                            .read<AuthCubit>();
                                                    if (authCubit
                                                            .allUserProfiles
                                                            .length >
                                                        1) {
                                                      while (Navigator.canPop(
                                                          context)) {
                                                        Navigator.pop(context);
                                                      }
                                                      authCubit
                                                          .switchToBusinessAccount();
                                                    } else {
                                                      while (Navigator.canPop(
                                                          context)) {
                                                        Navigator.pop(context);
                                                      }
                                                      context.openPage(
                                                        AppRoutes
                                                            .createBusinessProfile,
                                                      );
                                                    }
                                                  },
                                                  child: Container(
                                                    color: Colors.transparent,
                                                    child: Text(
                                                      'Бизнес-профиль →',
                                                      style: AppTexts()
                                                          .sp16
                                                          .copyWith(
                                                            color:
                                                                AppColors.link,
                                                          ),
                                                    ),
                                                  ),
                                                ),
                                              ],
                                            ),
                                          ),
                                        ],
                                        const SizedBox(height: 24),
                                      ],
                                    ),
                                    buttonTitle: 'Понятно',
                                    onSubmit: () {
                                      Navigator.pop(context);
                                    },
                                  )
                                      .then((value) {
                                    descriptionFocus.unfocus();
                                  });
                                },
                                child: SvgPicture.asset(
                                  AppIcons.info,
                                  height: 28,
                                ),
                              )
                            ],
                          ),
                          const SizedBox(height: 4),
                          Text(
                            'Укажите ссылку на карточку товара (одежда, обувь, аксессуары)',
                            style: AppTexts().sp16.copyWith(
                                  fontWeight: FontWeight.w400,
                                ),
                          ),
                        ],
                      ),
                    ),
                  ),
                  // SliverToBoxAdapter(child: SizedBox(height: 16)),
                  SliverPadding(
                    padding: const EdgeInsets.symmetric(
                        horizontal: 16, vertical: 16),
                    sliver: SliverGrid(
                      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
                        crossAxisCount: 2,
                        crossAxisSpacing: 8,
                        mainAxisSpacing: 12,
                        mainAxisExtent: MediaQuery.of(context).size.width / 2 -
                            16 +
                            36, //220,
                        // childAspectRatio: 250 /
                        //     (fashionModel?.products?.isEmpty == true ? 200 : 320),
                      ),
                      delegate: SliverChildBuilderDelegate(
                        (context, index) {
                          final products = fashionModel?.products;

                          if (products == null || products.isEmpty) {
                            // Первый товар - показать кнопку добавления
                            return product(null, false, index);
                          }

                          if (index < products.length) {
                            // Показать элемент из списка (реальный товар или null)
                            return product(products[index], false, index);
                          }

                          // Этот код не должен выполняться, так как childCount = products.length
                          return product(null, false, index);
                        },
                        childCount: () {
                          final products = fashionModel?.products;
                          if (products == null || products.isEmpty) {
                            return 1; // Показать пустую ячейку для добавления первого товара
                          }
                          // Всегда показываем ровно столько ячеек, сколько элементов в списке
                          // (включая null элемент в конце)
                          return products.length;
                        }(),
                      ),
                    ),
                  ),

                  // Не отображаем отступ только, если кол-во товаров равно 0 или 2 или 4
                  if (fashionModel?.products?.length == 0 ||
                      fashionModel?.products?.length == 2 ||
                      fashionModel?.products?.length == 4 ||
                      fashionModel?.products?.length == 6) ...[
                    const SliverToBoxAdapter(child: SizedBox(height: 36)),
                  ],
                  SliverToBoxAdapter(
                    child: SizedBox(
                      // height: 94,
                      child: Column(
                        children: [
                          Container(height: 0.5, color: AppColors.grey),
                          const SizedBox(height: 12),
                          CustomButton(
                            title: 'Опубликовать',
                            disabled: disaled(),
                            isLoading: state is CreateFashionLoading ||
                                state is FashionPublicateSuccess ||
                                state is FashionLoading,
                            onTap: () {
                              fashionCubit.updateFashion(
                                fashionModel: fashionModel,
                                disabled: disaled(),
                              );
                            },
                          ),
                          if (disaled()) ...[
                            const SizedBox(height: 12),
                            Padding(
                              padding:
                                  const EdgeInsets.symmetric(horizontal: 16),
                              child: Text(
                                disabledText,
                                style: AppTexts().sp14.copyWith(
                                      color: AppColors.secondary,
                                    ),
                              ),
                            ),
                          ],
                          const SizedBox(height: 34),
                        ],
                      ),
                    ),
                  ),
                ],
              ),
              if (state is FashionLoading || state is CreateFashionLoading)
                const LoadingOverlay(),
              // Container(
              //   color: AppColors.grey.withOpacity(0.5),
              //   child: const Center(
              //     child: SizedBox(
              //       height: 20,
              //       width: 20,
              //       child: CircularProgressIndicator(
              //         color: AppColors.primary,
              //       ),
              //     ),
              //   ),
              // )
            ],
          );
        }),
      ),
    );
  }

  bool disaled() {
    if (fashionModel == null) {
      return true;
    }
    if (textEditingController.text.length > 300) {
      return true;
    }
    if (fashionModel!.fashionImages == null) {
      return true;
    }
    if (fashionModel!.fashionImages!.length == 1) {
      return true;
    }
    if (fashionModel?.gender == null) {
      return true;
    }
    return false;
  }

  String get disabledText {
    if (fashionModel == null) {
      return 'Необходимо добавить фото образа';
    }
    if (textEditingController.text.length > 300) {
      return 'Максимум 300 символов в описании';
    }
    if (fashionModel!.fashionImages == null) {
      return 'Необходимо добавить фото образа';
    }
    if (fashionModel!.fashionImages!.length == 1 &&
        fashionModel?.gender == null) {
      return 'Необходимо добавить фото образа и выбрать пол';
    }
    if (fashionModel!.fashionImages!.length == 1) {
      return 'Необходимо добавить фото образа';
    }
    if (fashionModel?.gender == null) {
      return 'Необходимо выбрать пол образа';
    }
    return '';
  }

  Widget image(
    FashionImage? fashionImage,
    bool loading, {
    required FashionState fashionState,
  }) {
    double width = ((MediaQuery.of(context).size.width / 2));
    double height = (width / 9 * 16);
    return ClipRRect(
      borderRadius: BorderRadius.circular(12),
      child: SizedBox(
        width: width,
        height: height,
        // child: ,
        child: fashionImage?.photo == null
            ? GestureDetector(
                onTap: () {
                  HapticFeedback.mediumImpact();
                  if (fashionModel?.fashionImages?.length != 6) {
                    _pickImage();
                    descriptionFocus.unfocus();
                  }
                },
                child: Stack(
                  fit: StackFit.expand,
                  children: [
                    if (fashionState is FashionImageError) ...[
                      Image.file(
                        fashionState.file!,
                        height: height,
                        width: width,
                        fit: BoxFit.cover,
                      ),
                      Container(
                        height: height,
                        width: width,
                        // ignore: deprecated_member_use
                        color: Colors.black.withOpacity(0.4),
                        padding: const EdgeInsets.symmetric(horizontal: 12),
                        child: Column(
                          mainAxisAlignment: MainAxisAlignment.center,
                          children: [
                            SvgPicture.asset(
                              AppIcons.errorFashion,
                              height: 24,
                            ),
                            const SizedBox(height: 8),
                            Text(
                              'Фото не соответствует условиям',
                              style: AppTexts().sp14.copyWith(
                                    color: AppColors.white,
                                    height: 18 / 14,
                                  ),
                              textAlign: TextAlign.center,
                            ),
                          ],
                        ),
                      ),
                      Align(
                        alignment: Alignment.topRight,
                        child: GestureDetector(
                          onTap: () {
                            HapticFeedback.mediumImpact();

                            fashionCubit.closeBadPhoto();
                          },
                          child: Container(
                            height: 24,
                            width: 24,
                            margin: const EdgeInsets.only(top: 10, right: 10.5),
                            decoration: const BoxDecoration(
                              shape: BoxShape.circle,
                              color: AppColors.white25,
                            ),
                            alignment: Alignment.center,
                            child: SvgPicture.asset(
                              AppIcons.fashionClose,
                              height: 10,
                              width: 10,
                            ),
                          ),
                        ),
                      ),
                    ] else if (fashionModel?.fashionImages?.length != 6) ...[
                      SvgPicture.asset(AppIcons.addImage),
                      Padding(
                        padding: const EdgeInsets.symmetric(horizontal: 12),
                        child: Column(
                          mainAxisAlignment: MainAxisAlignment.center,
                          children: [
                            if (!loading) ...[
                              SvgPicture.asset(AppIcons.addCircle, height: 28),
                              const SizedBox(height: 8),
                              Text(
                                'Добавить фото',
                                style: AppTexts()
                                    .h4
                                    .copyWith(color: AppColors.primary),
                              ),
                              const SizedBox(height: 4),
                              Text(
                                'Размер менее 20 MB, формат 16:9',
                                style: AppTexts().sp14.copyWith(
                                      color: AppColors.description,
                                    ),
                                textAlign: TextAlign.center,
                              ),
                            ] else ...[
                              const Center(
                                child: SizedBox(
                                  height: 28,
                                  width: 28,
                                  child: CircularProgressIndicator(
                                    color: AppColors.primary,
                                  ),
                                ),
                              ),
                              const SizedBox(height: 8),
                              Text(
                                'Фото загружается',
                                style: AppTexts().sp14.copyWith(
                                      color: AppColors.description,
                                    ),
                                textAlign: TextAlign.center,
                              ),
                            ]
                          ],
                        ),
                      ),
                    ] else ...[
                      Container(
                        height: height,
                        width: width,
                        decoration: BoxDecoration(
                          borderRadius: BorderRadius.circular(12),
                          color: AppColors.grey,
                        ),
                        padding: const EdgeInsets.symmetric(horizontal: 13.25),
                        child: Column(
                          mainAxisAlignment: MainAxisAlignment.center,
                          children: [
                            SvgPicture.asset(
                              AppIcons.done,
                              height: 24,
                            ),
                            const SizedBox(height: 8),
                            Text(
                              'Добавлено максимальное количество фото образа (max 5)',
                              style: AppTexts().sp14.copyWith(
                                    color: AppColors.description,
                                    height: 18 / 14,
                                  ),
                              textAlign: TextAlign.center,
                            ),
                          ],
                        ),
                      )
                    ],
                  ],
                ),
              )
            : ClipRRect(
                borderRadius: BorderRadius.circular(12),
                child: Stack(
                  children: [
                    CachedNetworkImage(
                      height: height,
                      width: width,
                      imageUrl: Endpoints.mediaUrl + fashionImage!.photo,
                      placeholder: (context, url) => Stack(
                        fit: StackFit.expand,
                        children: [
                          SvgPicture.asset(AppIcons.addImage),
                          Column(
                            mainAxisAlignment: MainAxisAlignment.center,
                            children: [
                              const Center(
                                child: SizedBox(
                                  height: 28,
                                  width: 28,
                                  child: CircularProgressIndicator(
                                    color: AppColors.primary,
                                  ),
                                ),
                              ),
                              const SizedBox(height: 8),
                              Text(
                                'Фото загружается',
                                style: AppTexts().sp14.copyWith(
                                      color: AppColors.description,
                                    ),
                                textAlign: TextAlign.center,
                              ),
                            ],
                          ),
                        ],
                      ),
                      fit: BoxFit.cover,
                    ),
                    Align(
                      alignment: Alignment.topRight,
                      child: GestureDetector(
                        onTap: () {
                          HapticFeedback.mediumImpact();
                          if (fashionModel?.id != null) {
                            AppSheets(context).deleteDialog(
                              title: 'Удалить фото',
                              onDelete: () {
                                fashionCubit.deleteFashionImage(
                                  fashionId: fashionModel!.id!,
                                  photoId: fashionImage.id,
                                );
                              },
                            );
                          }
                        },
                        child: Container(
                          height: 40,
                          width: 40,
                          color: Colors.transparent,
                          margin: const EdgeInsets.all(2),
                          alignment: Alignment.center,
                          child: SvgPicture.asset(
                            AppIcons.closeOverlay,
                            height: 28,
                            width: 28,
                          ),
                        ),
                      ),
                    )
                  ],
                ),
              ),
      ),
    );
  }

  Widget product(Product? product, bool loading, int index) {
    double width = ((MediaQuery.of(context).size.width / 2));
    double height = (width / 9 * 16);

    // Отладочная информация
    if (kDebugMode) {
      print(
          '🛍️ Product widget - Index: $index, Product: ${product?.name ?? 'null'}, Photo: ${product?.photo ?? 'null'}, Total products: ${fashionModel?.products?.length ?? 0}');
    }

    return Container(
      width: width,
      height: height,
      decoration: BoxDecoration(
        borderRadius: BorderRadius.circular(12),
      ),
      // child: ,
      child: (product == null || product.photo == null)
          ? GestureDetector(
              onTap: () {
                HapticFeedback.mediumImpact();
                // _pickImage();
                // Проверяем количество реальных товаров (не null)
                final realProductCount =
                    fashionModel?.products?.where((p) => p != null).length ?? 0;
                if (realProductCount >= 5) {
                  return;
                }
                // if (product?.photo == null) {
                timer?.cancel();
                loadingTimer?.cancel();
                _pickProduct().then((_) {
                  print('State is reset');
                  context.read<FashionCubit>().resetUploadingState();
                  timer?.cancel();
                  loadingTimer?.cancel();
                  Future.delayed(const Duration(seconds: 3), () {
                    context.read<FashionCubit>().resetUploadingState();
                    timer?.cancel();
                    loadingTimer?.cancel();
                  });
                });
                descriptionFocus.unfocus();
                // } else if (product != null) {
                //   _showProduct(product);
                // }
              },
              child: Stack(
                fit: StackFit.expand,
                children: [
                  // Показываем заглушку если достигнуто максимальное количество товаров
                  if ((fashionModel?.products?.where((p) => p != null).length ??
                          0) >=
                      5) ...[
                    Align(
                      alignment: Alignment.topCenter,
                      child: Container(
                        width: (MediaQuery.of(context).size.width / 2) - 20,
                        height: (MediaQuery.of(context).size.width / 2) - 20,
                        decoration: BoxDecoration(
                          borderRadius: BorderRadius.circular(12),
                          color: AppColors.grey,
                        ),
                        padding: const EdgeInsets.symmetric(horizontal: 13.25),
                        child: Column(
                          mainAxisAlignment: MainAxisAlignment.center,
                          children: [
                            SvgPicture.asset(
                              AppIcons.done,
                              height: 24,
                            ),
                            const SizedBox(height: 8),
                            Text(
                              'Добавлено максимальное количество товаров  (max 5)',
                              style: AppTexts().sp14.copyWith(
                                    color: AppColors.description,
                                    height: 18 / 14,
                                  ),
                              textAlign: TextAlign.center,
                            ),
                          ],
                        ),
                      ),
                    )
                  ] else ...[
                    Align(
                      alignment: Alignment.topCenter,
                      child: SvgPicture.asset(
                        AppIcons.addImgProduct,
                        width: (MediaQuery.of(context).size.width / 2) - 20,
                        // width: 167.5,
                      ),
                    ),
                    Align(
                      alignment: Alignment.topCenter,
                      child: Container(
                        width: (MediaQuery.of(context).size.width / 2),
                        height: (MediaQuery.of(context).size.width / 2) - 20,
                        padding: const EdgeInsets.symmetric(horizontal: 8),
                        child: Column(
                          mainAxisAlignment: MainAxisAlignment.center,
                          children: [
                            if (!loading) ...[
                              SvgPicture.asset(
                                AppIcons.addCircle,
                                height: 28,
                              ),
                              const SizedBox(height: 8),
                              Text(
                                'Прикрепить товар',
                                style: AppTexts().h4.copyWith(
                                      color: AppColors.primary,
                                      fontWeight: FontWeight.bold,
                                    ),
                                textAlign: TextAlign.center,
                              ),
                              const SizedBox(height: 4),
                              Text(
                                'Ссылка на страницу товара',
                                style: AppTexts().sp14.copyWith(
                                      color: AppColors.description,
                                    ),
                                textAlign: TextAlign.center,
                              ),
                            ] else ...[
                              const Center(
                                child: SizedBox(
                                  height: 28,
                                  width: 28,
                                  child: CircularProgressIndicator(
                                    color: AppColors.primary,
                                  ),
                                ),
                              ),
                              const SizedBox(height: 8),
                              Text(
                                'Фото загружается',
                                style: AppTexts().sp14.copyWith(
                                      color: AppColors.description,
                                    ),
                                textAlign: TextAlign.center,
                              ),
                            ]
                          ],
                        ),
                      ),
                    )
                  ],
                ],
              ),
            )
          : GestureDetector(
              onTap: () {
                if (fashionModel?.products != null) {
                  // _showProduct(fashionModel!.products! ?? [], index);
                  descriptionFocus.unfocus();
                  AppSheets(context).showProduct(
                    fashionModel!.products ?? [],
                    index,
                    fashionModel?.createdDate,
                  );

                  // После возврата из модального окна проверяем и восстанавливаем null элемент
                  // Используем задержку для обработки после закрытия модального окна
                  Future.delayed(const Duration(milliseconds: 500), () {
                    if (mounted) {
                      setState(() {
                        _ensureNullElementAtEnd();
                      });
                    }
                  });
                }
              },
              child: Container(
                width: (MediaQuery.of(context).size.width / 2),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    Stack(
                      children: [
                        Container(
                          decoration: BoxDecoration(
                            borderRadius: BorderRadius.circular(12),
                            border:
                                Border.all(color: AppColors.grey, width: 0.5),
                          ),
                          child: ClipRRect(
                            borderRadius: BorderRadius.circular(12),
                            child: CachedNetworkImage(
                              width:
                                  (MediaQuery.of(context).size.width / 2) - 0,
                              height:
                                  (MediaQuery.of(context).size.width / 2) - 20,
                              fit: BoxFit.cover,
                              imageUrl:
                                  Endpoints.mediaUrl + (product.photo ?? ''),
                              placeholder: (context, url) => Stack(
                                fit: StackFit.expand,
                                children: [
                                  SvgPicture.asset(AppIcons.addImgProduct),
                                  Column(
                                    mainAxisAlignment: MainAxisAlignment.center,
                                    children: [
                                      const Center(
                                        child: SizedBox(
                                          height: 28,
                                          width: 28,
                                          child: CircularProgressIndicator(
                                            color: AppColors.primary,
                                          ),
                                        ),
                                      ),
                                      const SizedBox(height: 8),
                                      Text(
                                        'Фото загружается',
                                        style: AppTexts().sp14.copyWith(
                                              color: AppColors.description,
                                            ),
                                        textAlign: TextAlign.center,
                                      ),
                                    ],
                                  ),
                                ],
                              ),
                            ),
                          ),
                        ),
                        Align(
                          alignment: Alignment.topRight,
                          child: GestureDetector(
                            onTap: () {
                              HapticFeedback.mediumImpact();
                              if (fashionModel?.id != null) {
                                AppSheets(context).deleteDialog(
                                  title: 'Удалить товар',
                                  onDelete: () {
                                    fashionCubit.deleteProduct(
                                      fashionId: fashionModel!.id!,
                                      productId: product.id,
                                    );
                                  },
                                );
                              }
                            },
                            child: Container(
                              height: 40,
                              width: 40,
                              margin: const EdgeInsets.all(2),
                              alignment: Alignment.center,
                              child: SvgPicture.asset(
                                AppIcons.closeOverlay,
                                height: 28,
                                width: 28,
                              ),
                            ),
                          ),
                        )
                      ],
                    ),
                    const SizedBox(height: 7),
                    Flexible(
                      child: Text(
                        product.name,
                        maxLines: 1,
                        style: AppTexts().sp14.copyWith(
                              overflow: TextOverflow.ellipsis,
                            ),
                      ),
                    ),
                    Flexible(
                      child: formattedPrice(
                        product.price ?? '',
                        style: AppTexts().h4.copyWith(
                              fontWeight: FontWeight.bold,
                            ),
                      ),
                    ),
                  ],
                ),
              ),
            ),
    );
  }
}
```
