import 'dart:collection';

import 'package:flutter/foundation.dart';
import 'package:googleapis/youtube/v3.dart';
import 'package:http/http.dart' as http;

class FlutterDevPlaylists extends ChangeNotifier {
  FlutterDevPlaylists({
    required String flutterDevAccountId,
    required String youTubeApiKey,
  }) : _flutterDevAccountId = flutterDevAccountId {
    _api = YouTubeApi(
      _ApiKeyClient(
        client: http.Client(),
        key: youTubeApiKey,
      ),
    );
    _loadPlaylists();
  }

  Future<void> _loadPlaylists() async {
    String? nextPageToken;
    _playlists.clear();

    do {
      final response = await _api.playlists.list(
        ['snippet', 'contentDetails', 'id'],
        channelId: _flutterDevAccountId,
        maxResults: 50,
        pageToken: nextPageToken,
      );
      _playlists.addAll(response.items!);
      _playlists.sort((a, b) => a.snippet!.title!
          .toLowerCase()
          .compareTo(b.snippet!.title!.toLowerCase()));
      notifyListeners();
    } while (nextPageToken != null);
  }

  final String _flutterDevAccountId;
  late final YouTubeApi _api;

  final List<Playlist> _playlists = [];
  List<Playlist> get playlists => UnmodifiableListView(_playlists);

  final Map<String, List<PlaylistItem>> _playlistItems = {};
  List<PlaylistItem> playlistItems({required String playlistId}) {
    if (!_playlistItems.containsKey(playlistId)) {
      _playlistItems[playlistId] = [];
      _retrievePlaylist(playlistId);
    }
    return UnmodifiableListView(_playlistItems[playlistId]!);
  }

  Future<void> _retrievePlaylist(String playlistId) async {
    String? nextPageToken;
    do {
      var response = await _api.playlistItems.list(
        ['snippet', 'contentDetails'],
        playlistId: playlistId,
        maxResults: 25,
        pageToken: nextPageToken,
      );
      var items = response.items;
      if (items != null) {
        _playlistItems[playlistId]!.addAll(items);
      }
      notifyListeners();
      nextPageToken = response.nextPageToken;
    } while (nextPageToken != null);
  }
}

class _ApiKeyClient extends http.BaseClient {
  _ApiKeyClient({required this.key, required this.client});

  final String key;
  final http.Client client;

  @override
  Future<http.StreamedResponse> send(http.BaseRequest request) {
    final url = request.url.replace(queryParameters: {
      ...request.url.queryParametersAll,
      'key': [key]
    });

    return client.send(http.Request(request.method, url));
  }
}
Em seguida, adicione a página de detalhes da lista de reprodução individual.

lib / src / playlist_details.dart

import 'package:flutter/material.dart';
import 'package:googleapis/youtube/v3.dart';
import 'package:provider/provider.dart';
import 'package:url_launcher/link.dart';

import 'app_state.dart';

class PlaylistDetails extends StatelessWidget {
  const PlaylistDetails(
      {required this.playlistId, required this.playlistName, Key? key})
      : super(key: key);
  final String playlistId;
  final String playlistName;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(playlistName),
      ),
      body: Consumer<FlutterDevPlaylists>(
        builder: (context, flutterDev, _) {
          final playlistItems =
              flutterDev.playlistItems(playlistId: playlistId);
          if (playlistItems.isEmpty) {
            return const Center(child: CircularProgressIndicator());
          }

          return _PlaylistDetailsListView(playlistItems: playlistItems);
        },
      ),
    );
  }
}

class _PlaylistDetailsListView extends StatelessWidget {
  const _PlaylistDetailsListView({Key? key, required this.playlistItems})
      : super(key: key);
  final List<PlaylistItem> playlistItems;

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: playlistItems.length,
      itemBuilder: (context, index) {
        final playlistItem = playlistItems[index];
        return Padding(
          padding: const EdgeInsets.all(8.0),
          child: ClipRRect(
            borderRadius: BorderRadius.circular(4),
            child: Stack(
              alignment: Alignment.center,
              children: [
                Image.network(playlistItem.snippet!.thumbnails!.high!.url!),
                _buildGradient(context),
                _buildTitleAndSubtitle(context, playlistItem),
                _buildPlayButton(context, playlistItem),
              ],
            ),
          ),
        );
      },
    );
  }

  Widget _buildGradient(BuildContext context) {
    return Positioned.fill(
      child: DecoratedBox(
        decoration: BoxDecoration(
          gradient: LinearGradient(
            colors: [Colors.transparent, Theme.of(context).backgroundColor],
            begin: Alignment.topCenter,
            end: Alignment.bottomCenter,
            stops: const [0.5, 0.95],
          ),
        ),
      ),
    );
  }

  Widget _buildTitleAndSubtitle(
      BuildContext context, PlaylistItem playlistItem) {
    return Positioned(
      left: 20,
      right: 0,
      bottom: 20,
      child: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            playlistItem.snippet!.title!,
            style: Theme.of(context).textTheme.bodyText1!.copyWith(
                  fontSize: 18,
                  // fontWeight: FontWeight.bold,
                ),
          ),
          Text(
            playlistItem.snippet!.videoOwnerChannelTitle!,
            style: Theme.of(context).textTheme.bodyText2!.copyWith(
                  fontSize: 12,
                ),
          ),
        ],
      ),
    );
  }

  Widget _buildPlayButton(BuildContext context, PlaylistItem playlistItem) {
    return Container(
      width: 42,
      height: 42,
      decoration: const BoxDecoration(
        color: Colors.white,
        borderRadius: BorderRadius.all(
          Radius.circular(21),
        ),
      ),
      child: Center(
        child: Transform.scale(
          scale: 2,
          child: Link(
            uri: Uri.parse(
                'https://www.youtube.com/watch?v=${playlistItem.snippet!.resourceId!.videoId}'),
            builder: (context, followLink) => IconButton(
              onPressed: followLink,
              color: Colors.red,
              icon: const Icon(Icons.play_circle_fill),
            ),
          ),
        ),
      ),
    );
  }
}
  


